<script lang="ts">
	const J2J_VALUE = 10240000;
	const J2J_FOOTER_SIZE = 32;

	import * as CRC32 from 'crc-32/crc32c';
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
		let iv = new Uint8Array(J2J_VALUE);
		password = stringToBytes(password);

		let skip = false;
		let skipCount = 0;
		let passwordIndex = 0;

		for (let i = 0; i < J2J_VALUE; i++) {
			iv[i] = i % 256;
		}

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

			let footer = await file.slice(filesize - J2J_FOOTER_SIZE).arrayBuffer();
			let footer_info = parseFooter(footer);
			if (!footer_info) {
				alert('J2J 파일이 아닙니다.');
			}

			let blockCount = Math.max(Math.floor(J2J_VALUE / (filesize * 100)), 1);
			let iv = createIVWithPassword('qwerty');

			let stream = streamSaver.createWriteStream('wow.exe', {
				size: filesize - J2J_FOOTER_SIZE
			});
			
			const writer = stream.getWriter();

			for (let c = 0; c <= blockCount; c++) {
				let chunk = new Uint8Array(
					await file.slice(c * J2J_VALUE, c * J2J_VALUE + J2J_VALUE).arrayBuffer()
				);

				for (let i = 0; i < J2J_VALUE; i++) {
					chunk[i] ^= iv[i];
					iv[i] ^= chunk[i];
				}

				writer.write(chunk);
				chunk = null;
			}

			{
				let chunk = new Uint8Array(
					await file
						.slice(blockCount * J2J_VALUE, filesize - blockCount * J2J_VALUE - J2J_FOOTER_SIZE)
						.arrayBuffer()
				);
				writer.write(chunk);
				chunk = null;
			}

			for (let c = 0; c <= blockCount; c++) {
				let chunk = new Uint8Array(
					await file
						.slice(
							filesize - blockCount * J2J_VALUE - J2J_FOOTER_SIZE + c * J2J_VALUE,
							filesize - blockCount * J2J_VALUE - J2J_FOOTER_SIZE + c * J2J_VALUE + J2J_VALUE
						)
						.arrayBuffer()
				);

				for (let i = 0; i < J2J_VALUE; i++) {
					chunk[i] ^= iv[i];
					iv[i] ^= chunk[i];
				}
				chunk = null;
			}

			writer.close()
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
