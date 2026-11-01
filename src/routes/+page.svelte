<script>
	import {
		Button,
		Collection,
		List,
		ListRowImage,
		ListTextFieldRow,
		NavigationBar,
		NavigationBarTrailing
	} from 'apple-svelte';
	import './styles.css';

	let profileName = 'Custom Fonts';
	let profileOrganization = '';
	let profileDescription = 'Installs custom fonts';
	let fontFiles = [];
	let status = '';
	let busy = false;

	const colors = ['red', 'orange', 'yellow', 'green', 'mint', 'teal', 'cyan', 'blue', 'indigo', 'purple', 'pink', 'brown'];
	const color = colors[Math.floor(Math.random() * colors.length)];

	function updateFiles(event) {
		fontFiles = Array.from(event.currentTarget.files || []);
		status = fontFiles.length ? `${fontFiles.length} font file${fontFiles.length === 1 ? '' : 's'} selected` : '';
	}

	function xml(value) {
		return String(value ?? '').replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;').replace(/'/g, '&apos;');
	}

	function uuid() {
		return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
			const r = Math.random() * 16 | 0;
			return (c === 'x' ? r : (r & 3 | 8)).toString(16);
		}).toUpperCase();
	}

	function plistString(key, value) {
		return `<key>${key}</key><string>${xml(value)}</string>`;
	}

	async function buildProfile() {
		if (!fontFiles.length) throw new Error('Choose at least one .otf or .ttf font first.');
		if (!profileName.trim()) throw new Error('Enter a profile name.');
		const profileId = uuid();
		const payloads = [];
		for (const file of fontFiles) {
			const bytes = new Uint8Array(await file.arrayBuffer());
			let binary = '';
			const chunk = 0x8000;
			for (let i = 0; i < bytes.length; i += chunk) binary += String.fromCharCode(...bytes.subarray(i, i + chunk));
			const encoded = btoa(binary).match(/.{1,52}/g)?.join('\n\t\t\t') || '';
			const id = uuid();
			payloads.push(
				'<dict>' +
				'<key>Font</key><data>\n\t\t\t' + encoded + '\n\t\t</data>' +
				plistString('Name', file.name) +
				'<key>PayloadDescription</key><string>Configures Font settings</string>' +
				'<key>PayloadDisplayName</key><string>' + xml(file.name) + '</string>' +
				plistString('PayloadIdentifier', 'com.apple.font.' + id) +
				'<key>PayloadType</key><string>com.apple.font</string>' +
				plistString('PayloadUUID', id) +
				'<key>PayloadVersion</key><integer>1</integer>' +
				'</dict>'
			);
		}
		return '<?xml version="1.0" encoding="UTF-8"?>\n' +
			'<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">\n' +
			'<plist version="1.0"><dict>' +
			'<key>ConsentText</key><dict><key>default</key><string>' + xml(profileDescription) + '</string></dict>' +
			'<key>PayloadContent</key><array>' + payloads.join('') + '</array>' +
			plistString('PayloadDescription', profileDescription) +
			plistString('PayloadDisplayName', profileName.trim()) +
			plistString('PayloadIdentifier', 'com.fontinstaller.' + profileId) +
			plistString('PayloadOrganization', profileOrganization) +
			'<key>PayloadRemovalDisallowed</key><false/>' +
			plistString('PayloadType', 'Configuration') +
			plistString('PayloadUUID', profileId) +
			'<key>PayloadVersion</key><integer>1</integer>' +
			'</dict></plist>\n';
	}

	function safeFilename(name) {
		return name.replace(/[\\/:*?"<>|\x00-\x1F]/g, '_').replace(/\.+$/g, '') || 'font';
	}

	function downloadBlob(blob, filename) {
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = filename;
		a.style.display = 'none';
		document.body.appendChild(a);
		a.click();
		a.remove();
		setTimeout(() => URL.revokeObjectURL(url), 60000);
	}

	async function installProfile() {
		busy = true;
		status = 'Preparing installation profile…';
		try {
			const profile = await buildProfile();
			const blob = new Blob([profile], { type: 'application/x-apple-aspen-config' });
			const url = URL.createObjectURL(blob);
			status = 'Opening the profile. Follow the iOS prompt, then finish installation in Settings.';
			window.location.href = url;
			setTimeout(() => URL.revokeObjectURL(url), 120000);
		} catch (error) {
			status = error.message || 'Could not create the profile.';
		} finally {
			busy = false;
		}
	}

	function crc32(bytes) {
		let crc = -1;
		for (const byte of bytes) {
			crc ^= byte;
			for (let j = 0; j < 8; j++) crc = (crc >>> 1) ^ (crc & 1 ? 0xEDB88320 : 0);
		}
		return (crc ^ -1) >>> 0;
	}

	function zipStore(entries) {
		const encoder = new TextEncoder();
		const localParts = [];
		const centralParts = [];
		let offset = 0;
		const put16 = (v, n) => { v.push(n & 255, (n >>> 8) & 255); };
		const put32 = (v, n) => { v.push(n & 255, (n >>> 8) & 255, (n >>> 16) & 255, (n >>> 24) & 255); };
		for (const entry of entries) {
			const name = encoder.encode(entry.name);
			const data = entry.data instanceof Uint8Array ? entry.data : encoder.encode(entry.data);
			const crc = crc32(data);
			const local = [];
			put32(local, 0x04034b50); put16(local, 20); put16(local, 0x0800); put16(local, 0);
			put16(local, 0); put16(local, 0); put32(local, crc); put32(local, data.length); put32(local, data.length);
			put16(local, name.length); put16(local, 0);
			localParts.push(new Uint8Array(local), name, data);
			const central = [];
			put32(central, 0x02014b50); put16(central, 20); put16(central, 20); put16(central, 0x0800); put16(central, 0);
			put16(central, 0); put16(central, 0); put32(central, crc); put32(central, data.length); put32(central, data.length);
			put16(central, name.length); put16(central, 0); put16(central, 0); put16(central, 0); put16(central, 0); put32(central, 0); put32(central, offset);
			centralParts.push(new Uint8Array(central), name);
			offset += local.length + name.length + data.length;
		}
		const centralSize = centralParts.reduce((n, part) => n + part.length, 0);
		const end = [];
		put32(end, 0x06054b50); put16(end, 0); put16(end, 0); put16(end, entries.length); put16(end, entries.length);
		put32(end, centralSize); put32(end, offset); put16(end, 0);
		return new Blob([...localParts, ...centralParts, new Uint8Array(end)], { type: 'application/zip' });
	}

	async function downloadZip() {
		busy = true;
		status = 'Building ZIP…';
		try {
			const profile = await buildProfile();
			const entries = [{ name: safeFilename(profileName) + '.mobileconfig', data: profile }];
			for (const file of fontFiles) entries.push({ name: 'Fonts/' + safeFilename(file.name), data: new Uint8Array(await file.arrayBuffer()) });
			downloadBlob(zipStore(entries), safeFilename(profileName) + '.zip');
			status = `ZIP created with the combined profile and ${fontFiles.length} font file${fontFiles.length === 1 ? '' : 's'}.`;
		} catch (error) {
			status = error.message || 'Could not create the ZIP.';
		} finally {
			busy = false;
		}
	}
</script>

<svelte:head>
	<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Material+Symbols+Rounded:opsz,wght,FILL,GRAD@20..48,100..700,0..1,-50..200" />
	<meta name="apple-mobile-web-app-capable" content="yes" />
	<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
	<meta name="apple-mobile-web-app-title" content="Font Installer" />
	<meta name="description" content="Install multiple custom fonts on iOS and iPadOS using configuration profiles." />
	<title>Font Installer</title>
</svelte:head>

<main style="--colors-accent: var(--colors-{color}); --colors-accent-2: var(--colors-{color}-2)">
	<NavigationBar showBackground size="large" title="Font Installer" style="margin: -24px -16px 0px">
		<NavigationBarTrailing slot="leading" state="disabled" symbol="text_fields" />
		<NavigationBarTrailing slot="trailing-1" type="text" label="GitHub" onPress={() => window.open('https://github.com/MegaCode111REAL/ios-font-installer', '_blank')} />
	</NavigationBar>
	<div class="banner">
		<ListRowImage type="symbol" symbol="info" />
		<p>Choose one or more .otf or .ttf fonts. Install opens a combined iOS configuration profile; Download ZIP saves the profile and original font files together.</p>
	</div>
	<Collection style="row-gap: 24px">
		<div class="text">
			<div><p class="large-title symbol">info</p><h5 class="headline">About this tool</h5></div>
			<p>iOS and iPadOS install fonts through configuration profiles. You can include multiple fonts in one profile, then complete the installation in Settings.</p>
		</div>
		<div class="text">
			<div><p class="large-title symbol">match_case</p><h5 class="headline">Font support</h5></div>
			<p>Supports multiple .otf and .ttf files in a single profile. Collection fonts (.ttc) and variable fonts are not supported. Fonts are available in compatible apps that use Apple's font APIs.</p>
		</div>
		<div>
			<List showHeader header="PROFILE SETTINGS">
				<ListTextFieldRow bind:value={profileName} title="Name" />
				<ListTextFieldRow bind:value={profileOrganization} title="Organization" />
				<ListTextFieldRow bind:value={profileDescription} title="Description" />
			</List>
			<div class="file-picker">
				<label class="button file-button">
					<span class="symbol">attach_file</span>
					<span>{fontFiles.length ? `${fontFiles.length} font file${fontFiles.length === 1 ? '' : 's'} selected` : 'Choose font files'}</span>
					<input accept=".otf,.ttf,font/otf,font/ttf" class="hidden-input" on:change={updateFiles} type="file" multiple />
				</label>
				{#if fontFiles.length}
					<div class="selected-files">
						{#each fontFiles as file, index (file.name + file.size + index)}
							<div class="selected-file"><span class="symbol">description</span><span>{file.name}</span><span class="file-size">{(file.size / 1024).toFixed(0)} KB</span></div>
						{/each}
					</div>
				{/if}
				<div class="actions">
					<Button size="medium" type="filled" symbol="install_mobile" label="Install" onPress={installProfile} />
					<Button size="medium" type="gray" symbol="archive" label="Download ZIP" onPress={downloadZip} />
				</div>
				{#if busy}<p class="status" role="status">Working…</p>{:else if status}<p class="status" role="status">{status}</p>{/if}
			</div>
		</div>
	</Collection>
</main>

<style>
	main { display: flex; flex-direction: column; gap: 24px; padding: 24px 16px; }
	.banner { align-items: center; background: var(--colors-accent-2); border-radius: 10px; display: flex; gap: 12px; padding: 11px 16px; }
	.text { background: var(--bg-grouped-secondary); border-radius: 10px; display: flex; flex-direction: column; gap: 12px; padding: 11px 16px; }
	.symbol { color: var(--colors-accent); }
	.file-picker { display: flex; flex-direction: column; gap: 12px; margin-top: 14px; }
	.file-button { align-items: center; background: var(--colors-accent-2); border-radius: 40px; color: var(--colors-accent); display: inline-flex; gap: 8px; padding: 10px 16px; width: fit-content; max-width: 100%; }
	.file-button .symbol { font-family: var(--symbol-font-family); }
	.selected-files { background: var(--bg-grouped-secondary); border-radius: 10px; overflow: hidden; }
	.selected-file { align-items: center; border-bottom: 1px solid var(--separators-non-opaque); display: flex; gap: 10px; min-width: 0; padding: 10px 12px; }
	.selected-file:last-child { border-bottom: 0; }
	.selected-file > span:nth-child(2) { flex: 1; min-width: 0; overflow-wrap: anywhere; }
	.file-size { color: var(--labels-secondary); font-size: 12px; white-space: nowrap; }
	.actions { display: flex; flex-wrap: wrap; gap: 12px; }
	.status { color: var(--labels-secondary); font-size: 14px; line-height: 1.4; }
</style>
