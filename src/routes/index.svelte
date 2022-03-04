<script lang="ts">
	const CHUNK_SIZE = 1024000;
	const J2J_VALUE = 10240000;
	const J2J_FOOTER_SIZE = 32;
	
	import { onMount } from 'svelte';

	let streamSaver;
	onMount(async () => {
		streamSaver = await import('streamsaver');
	});

	let files: FileList;
	let fileInput: HTMLInputElement;

	function bytesToString(bytes) {
		return new TextDecoder().decode(bytes);
	}

	function stringToBytes(string) {
		return new TextEncoder().encode(string);
	}

	function onSelectClick() {
		fileInput && fileInput.click();
	}

	function parseFooter(footer) {
		footer = new Uint8Array(footer);

		if (footer.slice(0, 8).reduce((a, b) => a + b) != 0) {
			return;
		}

		let blockCount = footer[8];

		if (footer.slice(9, 16).reduce((a, b) => a + b) != 0) {
			return;
		}

		let crc = parseInt(bytesToString(footer.slice(16, 24)), 16);

		if (bytesToString(footer.slice(24, 32)) != 'L3000009') {
			return;
		}

		return {
			blockCount,
			crc
		};
	}

	function createIV() {
		let iv = new Uint8Array(J2J_VALUE);
		for (let i = 0; i <= J2J_VALUE; i++) {
			iv[i] = i % 256;
		}
		return iv;
	}

	function createIVWithPassword(password) {
		let iv = createIV();
		password = stringToBytes(password);

		let skip = false;
		let skipCount = 0;
		let passwordIndex = 0;

		for (let i = 0; i < J2J_VALUE; i++) {
			if (skip) {
				skip = false;
				skipCount++;
				continue;
			}

			if (skipCount >= password.length) {
				skipCount = 0;
				i += 1;
			}

			skip = true;

			iv[i] += password[passwordIndex++];
			if (passwordIndex >= password.length) {
				passwordIndex = 0;
			}
		}
		return iv;
	}

	async function onSelectedFile() {
		if (files && files.length > 0) {
			let file = files[0];
			let filesize = file.size;
			let filename = file.name;

			let footer = await file.slice(filesize - J2J_FOOTER_SIZE).arrayBuffer();
			let footer_info = parseFooter(footer);
			if (!footer_info) {
				alert('J2J 파일이 아닙니다.');
				return;
			}

			let password = prompt('비밀번호를 입력해주세요');
			if (password == null) {
				alert('취소하셨습니다.');
				return;
			}
			let blockCount = Math.max(Math.floor(J2J_VALUE / (filesize * 100)), 1);

			let iv: Uint8Array;
			if (password == '') {
				iv = createIV();
			} else {
				iv = createIVWithPassword(password);
			}

			let outputName: string;
			if (filename.endsWith('.j2j') || filename.endsWith('.J2J')) {
				outputName = filename.substring(0, filename.length - 4);
			} else {
				outputName = filename + '.dec';
			}

			let stream = streamSaver.createWriteStream(outputName, {
				size: filesize - J2J_FOOTER_SIZE
			});

			const writer = stream.getWriter();
			for (let c = 0; c < blockCount; c++) {
				let start = c * J2J_VALUE;
				let end = start + J2J_VALUE;
				let remain = end - start;
				
				let idx = 0;
				while (remain > 0) {
					let ee = start + CHUNK_SIZE;
					if (ee > end) {
						ee = end;
					}
					let chunk = new Uint8Array(await file.slice(start, ee).arrayBuffer());
					for (let i = 0; i < chunk.byteLength; i++) {
						chunk[i] ^= iv[idx];
						iv[idx] ^= chunk[i];
						idx += 1;
					}
					start += chunk.byteLength;
					remain -= chunk.byteLength
					await writer.write(chunk);
				}
			}

			{
				let start = blockCount * J2J_VALUE;
				let end = filesize - blockCount * J2J_VALUE - J2J_FOOTER_SIZE;
				let remain = end - start;
				
				while (remain > 0) {
					let ee = start + CHUNK_SIZE;
					if (ee > end) {
						ee = end;
					}
					let chunk = new Uint8Array(await file.slice(start, ee).arrayBuffer());
					remain -= ee - start;
					start += chunk.byteLength;
					await writer.write(chunk);
				}
			}

			for (let c = 0; c < blockCount; c++) {
				let start = filesize - blockCount * J2J_VALUE - J2J_FOOTER_SIZE + c * J2J_VALUE;
				let end = filesize - blockCount * J2J_VALUE - J2J_FOOTER_SIZE + c * J2J_VALUE + J2J_VALUE;
				let remain = end - start;
				
				let idx = 0;
				while (remain > 0) {
					let ee = start + CHUNK_SIZE;
					if (ee > end) {
						ee = end;
					}
					let chunk = new Uint8Array(await file.slice(start, ee).arrayBuffer());
					for (let i = 0; i < chunk.byteLength; i++) {
						chunk[i] ^= iv[idx];
						iv[idx] ^= chunk[i];
						idx += 1;
					}
					start += chunk.byteLength;
					remain -= chunk.byteLength
					await writer.write(chunk);
				}
			}
			writer.close();
		}
	}
</script>

<svelte:head>
	<script
		src="https://cdn.jsdelivr.net/npm/web-streams-polyfill@2.0.2/dist/ponyfill.min.js"></script>
	<script src="https://cdn.jsdelivr.net/npm/streamsaver@2.0.3/StreamSaver.min.js"></script>
</svelte:head>

<div class="w-screen h-screen flex items-center justify-center flex-col select-none">
	<div class="rounded border border-gray-300 px-14 py-10 cursor-pointer" on:click={onSelectClick}>
		<input class="hidden" type="file" bind:this={fileInput} bind:files on:change={onSelectedFile} />
		<img class="w-14 h-auto py-1" src="file-text.svg" alt="file_icon" />
		<span>선택하기</span>
	</div>
</div>
