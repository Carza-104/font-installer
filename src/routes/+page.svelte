<script>
	import {
		Button,
		Collection,
		List,
		ListTextFieldRow,
		NavigationBar,
		NavigationBarTrailing
	} from 'apple-svelte';
	import './styles.css';

	let base64EncodedFontFile = undefined;
	let blob = undefined;
	let fontFile = undefined;
	let fontFilename = undefined;
	let link = undefined;
	let profile = undefined;
	let profileDescription = undefined;
	let profileFilename = undefined;
	let profileName = undefined;
	let profileOrganization = undefined;
	let reader = undefined;
	let url = undefined;
	let uuid = undefined;

	function updateButtonText() {
		if (document.getElementById('font-file').files[0] === undefined) {
			return;
		}
		fontFile = document.getElementById('font-file').files[0];
		fontFilename = fontFile.name;
		document.getElementById('upload-button').textContent = fontFilename;
	}

	function init() {
		if (document.getElementById('font-file').files[0] === undefined) {
			return;
		}
		profileName = document.getElementById('profile-name').value;
		profileOrganization = document.getElementById('profile-organization').value;
		profileDescription = document.getElementById('profile-description').value;
		fontFile = document.getElementById('font-file').files[0];
		fontFilename = fontFile.name;
		profileFilename = profileName + '.mobileconfig';

		uuid = [generateUUID(), generateUUID(), generateUUID()];

		/* URL.createObjectURL uses UUIDs to generate unique URLs. */
		function generateUUID() {
			blob = new Blob();
			url = URL.createObjectURL(blob);
			/* Remove the URL. This leaves us with a full UUID with RFC 4122 version 4 random bytes. */
			uuid = url.replace(/.*\//, '').toUpperCase();
			return uuid;
		}

		base64EncodeFontFile(fontFile);

		function base64EncodeFontFile(file) {
			reader = new FileReader();
			reader.readAsDataURL(file);
			reader.addEventListener(
				'load',
				() => {
					/* Remove "data:font/otf;base64," or "data:font/ttf;base64,". */
					/* Separate the string in groups of 52 characters for better consistency with Apple Configurator. */
					/* Concatenate the strings with a line break and 3 tabs. */
					base64EncodedFontFile = reader.result
						.slice(21)
						.match(/.{1,52}/g)
						.join('\n\t\t\t');

					/* Use string interpolation instead of multi-line strings to prevent it from messing with the indentation. */
					profile =
						'<?xml version="1.0" encoding="UTF-8"?>\n' +
						'<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">\n' +
						'<plist version="1.0">\n' +
						'<dict>\n' +
						'\t<key>ConsentText</key>\n' +
						'\t<dict>\n' +
						'\t\t<key>default</key>\n' +
						'\t\t<string>' +
						profileDescription +
						'</string>\n' +
						'\t</dict>\n' +
						'\t<key>PayloadContent</key>\n' +
						'\t<array>\n' +
						'\t\t<dict>\n' +
						'\t\t\t<key>Font</key>\n' +
						'\t\t\t<data>\n' +
						'\t\t\t' +
						base64EncodedFontFile +
						'\n' +
						'\t\t\t</data>\n' +
						'\t\t\t<key>Name</key>\n' +
						'\t\t\t<string>' +
						fontFilename +
						'</string>\n' +
						'\t\t\t<key>PayloadDescription</key>\n' +
						'\t\t\t<string>Configures Font settings</string>\n' +
						'\t\t\t<key>PayloadDisplayName</key>\n' +
						'\t\t\t<string>Fonts</string>\n' +
						'\t\t\t<key>PayloadIdentifier</key>\n' +
						'\t\t\t<string>com.apple.font.' +
						uuid[0] +
						'</string>\n' +
						'\t\t\t<key>PayloadType</key>\n' +
						'\t\t\t<string>com.apple.font</string>\n' +
						'\t\t\t<key>PayloadUUID</key>\n' +
						'\t\t\t<string>' +
						uuid[0] +
						'</string>\n' +
						'\t\t\t<key>PayloadVersion</key>\n' +
						'\t\t\t<integer>1</integer>\n' +
						'\t\t</dict>\n' +
						'\t</array>\n' +
						'\t<key>PayloadDescription</key>\n' +
						'\t<string>' +
						profileDescription +
						'</string>\n' +
						'\t<key>PayloadDisplayName</key>\n' +
						'\t<string>' +
						profileName +
						'</string>\n' +
						'\t<key>PayloadIdentifier</key>\n' +
						'\t<string>Toms-MacBook-Air.' +
						uuid[1] +
						'</string>\n' +
						'\t<key>PayloadOrganization</key>\n' +
						'\t<string>' +
						profileOrganization +
						'</string>\n' +
						'\t<key>PayloadRemovalDisallowed</key>\n' +
						'\t<false/>\n' +
						'\t<key>PayloadType</key>\n' +
						'\t<string>Configuration</string>\n' +
						'\t<key>PayloadUUID</key>\n' +
						'\t<string>' +
						uuid[2] +
						'</string>\n' +
						'\t<key>PayloadVersion</key>\n' +
						'\t<integer>1</integer>\n' +
						'</dict>\n' +
						'</plist>\n';

					blob = new Blob([profile], { type: 'application/x-apple-aspen-config' });
					url = URL.createObjectURL(blob);
					link = document.createElement('a');
					link.download = profileFilename;
					link.href = url;
					link.click();
				},
				false
			);
		}
	}

	/* Randomly generate the accent color. */
	let colors = [
		'red',
		'orange',
		'yellow',
		'green',
		'mint',
		'teal',
		'cyan',
		'blue',
		'indigo',
		'purple',
		'pink',
		'brown'
	];

	let randomIndex = Math.floor(Math.random() * colors.length);

	let color = colors[randomIndex];
</script>

<svelte:head>
	<link
		rel="stylesheet"
		href="https://fonts.googleapis.com/css2?family=Material+Symbols+Rounded:opsz,wght,FILL,GRAD@20..48,100..700,0..1,-50..200"
	/>
	<meta name="apple-mobile-web-app-capable" content="yes" />
	<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
	<meta name="apple-mobile-web-app-title" content="Font Installer" />
	<meta
		name="description"
		content="This tool installs fonts on Apple devices using configuration profiles."
	/>
	<title>Font Installer</title>
</svelte:head>
<main style="--colors-accent: var(--colors-{color}); --colors-accent-2: var(--colors-{color}-2)">
	<NavigationBar showBackground size="large" title="Font Installer" style="margin: -24px -16px 0px">
		<NavigationBarTrailing slot="leading" state="disabled" symbol="pending" />
		<NavigationBarTrailing
			slot="trailing-1"
			type="text"
			label="GitHub"
			onPress={() => window.open('https://github.com/Carza-104/font-installer', '_blank')}
		/>
	</NavigationBar>
	<div class="banner">
		This page was built with apple-svelte, a Svelte component library based on Apple's UI design
		language. Check it out on <a href="https://github.com/Carza-104/apple-svelte">GitHub</a>.
	</div>
	<Collection style="gap: 24px">
		<div class="text">
			<div>
				<p class="large-title symbol">info</p>
				<h5 class="headline">About this tool</h5>
			</div>
			<p>
				This tool installs fonts systemwide on Apple devices using configuration profiles,
				.mobileconfig XML files.
			</p>
		</div>
		<div class="text">
			<div>
				<p class="large-title symbol">match_case</p>
				<h5 class="headline">Font support</h5>
			</div>
			<p>
				Each profile must contain one font. Collection formats (.otf and .ttc) and variable fonts
				are not supported.
			</p>
		</div>
		<div class="text">
			<div>
				<p class="large-title symbol">construction</p>
				<h5 class="headline">Accessing fonts</h5>
			</div>
			<p>
				Fonts are available to apps using the Font APIs introduced in iOS 13, like Pages, Numbers,
				and Keynote. You can see the profile in Settings.
			</p>
		</div>
		<div>
			<List showHeader header="PROFILE INSTALLER">
				<ListTextFieldRow title="Name" id="profile-name" />
				<ListTextFieldRow title="Org…" id="profile-organization" />
				<ListTextFieldRow title="Desc…" id="profile-description" />
			</List>
			<div class="buttons">
				<label>
					<input
						accept=".otf,.ttf"
						class="hidden-input"
						id="font-file"
						on:change={updateButtonText}
						type="file"
					/>
					<div class="button">
						<p class="symbol">north</p>
						<p class="subheadline" id="upload-button">Font</p>
					</div>
				</label>
				<Button size="medium" type="filled" symbol="south" label="Download" onPress={init} />
			</div>
		</div>
	</Collection>
</main>

<style>
	main {
		display: flex;
		flex-direction: column;
		gap: 24px;
		padding: 24px 16px;
	}

	.banner {
		background: var(--colors-accent-2);
		border-radius: 10px;
		padding: 11px 16px;
	}

	.text {
		background: var(--bg-grouped-secondary);
		border-radius: 10px;
		display: flex;
		flex-direction: column;
		gap: 12px;
		padding: 11px 16px;
	}

	.symbol {
		color: var(--colors-accent);
	}

	.buttons {
		display: flex;
		gap: 12px;
	}

	.button {
		align-items: center;
		background: var(--colors-accent-2);
		border-radius: 40px;
		display: flex;
		color: var(--colors-accent);
		gap: 4px;
		padding: 7px 14px;
		white-space: pre;
		width: min-content;
	}

	.button .symbol {
		color: var(--colors-accent);
		font-family: var(--symbol-font-family);
		font-size: calc(15px * var(--medium-symbol-font-size-multiplier));
		font-weight: calc(400 / var(--medium-symbol-font-size-multiplier));
		line-height: 20px;
	}
</style>
