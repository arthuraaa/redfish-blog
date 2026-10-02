---
title: "Browser Fingerprint"
date: 2025-01-01
draft: false
---

<style>
    .fp-tool, .fp-tool * {margin: 0; padding: 0; box-sizing: border-box;}
    .fp-tool {background: #0a0a0a; color: #aaa; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Arial, sans-serif; line-height: 1.6; max-width: 1200px; margin: 0 auto; padding: 40px 20px; border-radius: 8px;}
    .fp-tool h1 {color: #00ff00; margin-bottom: 10px; font-size: 32px;}
    .fp-tool .fp-info {color: #888; font-size: 0.95em; margin-bottom: 20px;}
    .fp-tool button {padding: 10px 20px; background: #00aa00; color: white; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; transition: all 0.2s ease; margin-right: 10px; margin-bottom: 10px;}
    .fp-tool button:hover {transform: scale(1.05); opacity: 0.9;}
    .fp-tool button.fp-copy-btn {background: #0066cc;}
    .fp-tool .fp-row {background: #161616; border: 1px solid #333; border-radius: 6px; margin-bottom: 10px; padding: 12px 16px;}
    .fp-tool .fp-name {color: #00ff00; font-family: monospace; font-weight: bold; margin-bottom: 6px; font-size: 14px;}
    .fp-tool .fp-value {background: #0a0a0a; border: 1px solid #2a2a2a; padding: 10px; border-radius: 4px; font-family: "Courier New", monospace; font-size: 12px; color: #88ff88; white-space: pre-wrap; word-break: break-all; max-height: 240px; overflow-y: auto;}
    .fp-tool #fp-status {color: #888; margin: 12px 0 20px 0; font-size: 0.9em;}
</style>

<div class="fp-tool">
    <p class="fp-info">Comme <b>AmIunique</b> : cette page collecte les attributs que votre navigateur expose volontairement ou non (canvas, WebGL, audio, polices, capteurs...). Aucune donnée n'est envoyée sur un serveur : tout reste dans votre navigateur, vous pouvez le vérifier dans le code source.</p>
    <button onclick="runFingerprint()">🔄 Re-run</button>
    <button class="fp-copy-btn" id="fp-copy-btn" onclick="copyFingerprint()">📋 Copy JSON</button>
    <div id="fp-status">⏳ Collecte des données...</div>
    <div id="fp-results"></div>
</div>

<div class="fp-tool">
<p class="fp-info">Informations déduites de votre adresse IP publique (requête vers un service de géolocalisation IP) et, si vous l'autorisez, de la géolocalisation GPS de votre appareil. C'est ce qu'un site quelconque peut déduire sur vous sans aucun script intrusif.</p>
<div id="geo-status">⏳ Résolution de l'IP publique...</div>
<div id="ip-results"></div>
<button onclick="requestGps()">📍 Révéler ma position GPS précise</button>
<div id="gps-results"></div>
</div>

<script>
(function() {
    const browserInfo = {};

    // 1 - User agent
    browserInfo.userAgent = navigator.userAgent;

    // 2 - Accept header
    browserInfo.acceptHeader = navigator.language;

    // 3 - Content encoding
    browserInfo.contentEncoding = 'gzip, deflate, br'; // Typically set by server

    // 4 - Content language
    browserInfo.contentLanguage = navigator.language;

    // 5 - Upgrade Insecure Requests
    browserInfo.upgradeInsecureRequests = '1'; // Typically set by browser

    // 6 - Cache control
    browserInfo.cacheControl = 'no-cache'; // Typically set by browser

    // 7 - Referer
    browserInfo.referer = document.referrer;

    // JavaScript attributes
    browserInfo.jsAttributes = {
        // 1 - User agent
        userAgent: navigator.userAgent,

        // 2 - Platform
        platform: navigator.platform,

        // 3 - Cookies enabled
        cookiesEnabled: navigator.cookieEnabled,

        // 4 - Timezone
        timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,

        // 5 - Content language
        contentLanguage: navigator.language,

        // 6 - Canvas (fingerprint)
        canvas: (() => {
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            ctx.textBaseline = 'top';
            ctx.font = '14px Arial';
            ctx.textBaseline = 'alphabetic';
            ctx.fillStyle = '#f60';
            ctx.fillRect(125, 1, 62, 20);
            ctx.fillStyle = '#069';
            ctx.fillText('Hello, world!', 2, 15);
            ctx.fillStyle = 'rgba(102, 204, 0, 0.7)';
            ctx.fillText('Hello, world!', 4, 17);
            return canvas.toDataURL();
        })(),

        // 7 - List of fonts (JS)
        fonts: (() => {
            const fontList = ['cursive', 'monospace', 'serif', 'sans-serif', 'fantasy', 'default', 'Arial', 'Arial Black', 'Arial Narrow', 'Arial Rounded MT Bold', 'Book Antiqua', 'Bookman Old Style', 'Bradley Hand ITC', 'Bodoni MT', 'Calibri', 'Century', 'Century Gothic', 'Casual', 'Comic Sans MS', 'Consolas', 'Copperplate Gothic Bold', 'Courier', 'Courier New', 'English Text MT', 'Felix Titling', 'Futura', 'Garamond', 'Geneva', 'Georgia', 'Gentium', 'Haettenschweiler', 'Helvetica', 'Impact', 'Jokerman', 'King', 'Kootenay', 'Latha', 'Liberation Serif', 'Lucida Console', 'Lalit', 'Lucida Grande', 'Magneto', 'Mistral', 'Modena', 'Monotype Corsiva', 'MV Boli', 'OCR A Extended', 'Onyx', 'Palatino Linotype', 'Papyrus', 'Parchment', 'Pericles', 'Playbill', 'Segoe Print', 'Shruti', 'Tahoma', 'TeX', 'Times', 'Times New Roman', 'Trebuchet MS', 'Verdana', 'Verona', 'Arial Cyr', 'Comic Sans MS', 'Arial Black', 'Chiller', 'Arial Narrow', 'Arial Rounded MT Bold', 'Baskerville Old Face', 'Berlin Sans FB', 'Blackadder ITC', 'Lucida Console', 'Symbol', 'Times New Roman', 'Webdings', 'Agency FB', 'Vijaya', 'Algerian', 'Arial Unicode MS', 'Bodoni MT Poster Compressed', 'Bookshelf Symbol 7', 'Calibri', 'Cambria', 'Cambria Math', 'Kartika', 'MS Mincho', 'MS Outlook', 'MT Extra', 'Segoe UI', 'Aharoni', 'Aparajita', 'Amienne', 'cursive', 'Academy Engraved LET', 'LCD', 'LuzSans-Book', 'sans-serif', 'ZWAdobeF', 'Eurostile', 'SimSun-PUA', 'Blackletter686 BT', 'Myriad Web Pro Condensed', 'Matisse ITC', 'Bell Gothic Std Black', 'David Transparent', 'Adobe Caslon Pro', 'AR BERKLEY', 'Australian Sunrise', 'Myriad Web Pro', 'Gentium Basic', 'Highlight LET', 'Adobe Myungjo Std M', 'GothicE', 'HP PSG', 'DejaVu Sans', 'Arno Pro', 'Futura Bk', 'DejaVu Sans Condensed', 'Euro Sign', 'Neurochrome', 'Bell Gothic Std Light', 'Jokerman Alts LET', 'Adobe Fan Heiti Std B', 'Baby Kruffy', 'Tubular', 'Woodcut', 'HGHeiseiKakugothictaiW3', 'YD2002', 'Tahoma Small Cap', 'Helsinki', 'Bickley Script', 'Unicorn', 'X-Files', 'GENISO', 'Frutiger SAIN Bd v.1', 'Opus', 'ZDingbats', 'ABSALOM', 'Vagabond', 'Year supply of fairy cakes', 'Myriad Condensed Web', 'Segoe Media Center', 'Coronet', 'Helsinki Metronome', 'Segoe Condensed', 'Weltron Urban', 'AcadEref', 'DecoType Naskh', 'Freehand521 BT', 'Opus Chords Sans', 'Enviro', 'SWGamekeys MT', 'Croobie', 'Arial Narrow Special G1', 'AVGmdBU', 'Candles', 'Futura Bk BT', 'Andy', 'QuickType', 'WP Arabic Sihafa', 'DigifaceWide', 'ELEGANCE', 'BRAZIL', 'Pepita MT', 'Nina', 'Geneva', 'OCR B MT', 'Futura', 'Blade Runner Movie Font', 'Allegro BT', 'Lucida Blackletter', 'AGA Arabesque', 'AdLib BT', 'Clarendon', 'Monotype Sorts', 'Alibi', 'Bremen Bd BT', 'mono', 'News Gothic MT', 'AvantGarde Bk BT', 'chs_boot', 'fantasy', 'Palatino', 'BernhardFashion BT', 'Courier New', 'CloisterBlack BT', 'Scriptina', 'Tahoma', 'BernhardMod BT', 'Virtual DJ', 'Nokia Smiley', 'Boulder', 'Andale Mono IPA', 'Belwe Lt BT', 'Calligrapher', 'Belwe Cn BT', 'Tanseek Pro Arabic', 'FuturaBlack BT', 'Abadi MT Condensed', 'Mangal', 'Chaucer', 'Belwe Bd BT', 'Liberation Serif', 'DomCasual BT', 'Bitstream Vera Sans', 'URW Gothic L', 'GeoSlab703 Lt BT', 'Bitstream Vera Sans Mono', 'Nimbus Mono L', 'Heather', 'Antique Olive', 'Clarendon Cn BT', 'Amazone BT', 'Bitstream Vera Serif', 'Utopia', 'Americana BT', 'Map Symbols', 'Bitstream Charter', 'Aurora Cn BT', 'CG Omega', 'Lohit Punjabi', 'Balloon XBd BT', 'Akhbar MT', 'Courier 10 Pitch', 'Benguiat Bk BT', 'Market', 'Cursor', 'Bodoni Bk BT', 'Letter Gothic', 'Luxi Sans', 'Brush455 BT', 'Sydnie', 'Lohit Hindi', 'Lithograph', 'Albertus', 'DejaVu LGC Serif', 'Lydian BT', 'Antique Olive Compact', 'KacstArt', 'Incised901 Bd BT', 'Clarendon Extended', 'Lohit Telugu', 'Incised901 Lt BT', 'GiovanniITCTT', 'KacstOneFixed', 'Folio XBd BT', 'Edda', 'Loma', 'Formal436 BT', 'Fine Hand', 'Garuda', 'Impress BT', 'RefSpecialty', 'Sazanami Mincho', 'Staccato555 BT', 'VL Gothic', 'Hkmer OS', 'WP BoxDrawing', 'Clarendon Blk BT', 'Droid Sans', 'CommonBullets', 'Sherwood', 'Helvetica', 'CopprplGoth Bd BT', 'Smudger Alts LET', 'BPG Rioni', 'CopprplGoth BT', 'Guitar Pro 5', 'Estrangelo TurAbdin', 'Dauphin', 'Arial Tur', 'English111 Vivace BT', 'Steamer', 'OzHandicraft BT', 'Futura Lt BT', 'Liberation Sans Narrow', 'Futura XBlk BT', 'Candy Round BTN Cond', 'GoudyHandtooled BT', 'GrilledCheese BTN Cn', 'GoudyOlSt BT', 'Galeforce BTN', 'Kabel Bk BT', 'Sneakerhead BTN Shadow', 'OCR-A BT', 'Denmark', 'OCR-B 10 BT', 'Swiss921 BT', 'PosterBodoni BT', 'Arial (Arabic)', 'Serifa BT', 'FlemishScript BT', 'Arial', 'American Typewriter', 'Arial Black', 'Apple Symbols', 'Arial Narrow', 'AppleMyungjo', 'Arial Rounded MT Bold', 'Zapfino', 'Arial Unicode MS', 'BlairMdITC TT-Medium', 'Century Gothic', 'Cracked', 'Papyrus', 'KufiStandardGK', 'Plantagenet Cherokee', 'Courier', 'Helvetica', 'Baskerville Old Face', 'Apple Casual', 'Type Embellishments One LET', 'Bookshelf Symbol 7', 'Abadi MT Condensed Extra Bold', 'Calibri', 'Calibri Bold', 'Calisto MT', 'Chalkduster', 'Cambria', 'Franklin Gothic Book Italic', 'Century', 'Geneva CY', 'Franklin Gothic Book', 'Helvetica Light', 'Gill Sans MT', 'Academy Engraved LET', 'MT Extra', 'Bank Gothic', 'Eurostile', 'Bodoni SvtyTwo SC ITC TT-Book', 'Tekton Pro', 'Courier CE', 'Maestro', 'BO Futura BoldOblique', 'Lucida Bright Demibold', 'New', 'AGaramond', 'Charcoal', 'DIN-Black', 'Lucida Sans Demibold', 'Stone Sans OS ITC TT-Bold', 'AGaramond Italic', 'Bickham Script Pro Regular', 'Adobe Arabic Bold', 'AGaramond Semibold', 'Al Bayan Bold', 'Doremi', 'AGaramond SemiboldItalic', 'Arno Pro Bold', 'Casual', 'B Futura Bold', 'Frutiger 47LightCn', 'Gadget', 'HelveticaNeueLT Std Bold', 'Frutiger 57Cn', 'DejaVu Serif Italic Condensed', 'Myriad Pro Black It', 'Frutiger 67BoldCn', 'Gentium Basic Bold', 'Sand', 'GillSans', 'H Futura Heavy', 'Liberation Mono Bold', 'GillSans Bold', 'Cambria Math', 'Courier Final Draft', 'HelveticaNeue BlackCond', 'cursive', 'Techno', 'HelveticaNeue BlackCondObl', 'Gabriola', 'JazzText Extended', 'HelveticaNeue BlackExt', 'sans-serif', 'Textile', 'HelveticaNeue BlackExtObl fantasy', 'HelveticaNeue BoldCond', 'Palatino Linotype Bold', 'HelveticaNeue BoldCondObl', 'BIRTH OF A HERO', 'HelveticaNeue BoldExt', 'Bleeding Cowboys', 'HelveticaNeue BoldExtObl', 'ChopinScript', 'HelveticaNeue ExtBlackCond', 'LCD', 'HelveticaNeue ExtBlackCondObl', 'Myriad Web Pro Condensed', 'HelveticaNeue HeavyCond', 'Scriptina', 'HelveticaNeue HeavyCondObl', 'OpenSymbol', 'HelveticaNeue HeavyExt', 'Virtual DJ', 'HelveticaNeue HeavyExtObl', 'Guitar Pro 5', 'HelveticaNeue LightCondObl', 'Nueva Std', 'HelveticaNeue ThinCond', 'Chicago', 'HelveticaNeue ThinCondObl', 'Nueva Std Bold', 'Brush Script MT', 'Capitals', 'Myriad Web Pro', 'Avant Garde', 'B Avant Garde Demi', 'Nueva Std Bold Italic', 'BI Avant Garde DemiOblique', 'MaestroTimes', 'Univers BoldExtObl', 'APC Courier', 'Myriad Web Pro Bold', 'Liberation Serif', 'Myriad Pro Light', 'Carta', 'DIN-Bold', 'DIN-Light', 'Myriad Web Pro Condensed Italic', 'DIN-Medium', 'Tekton Pro Oblique', 'DIN-Regular', 'AScore', 'HelveticaNeue UltraLigCondObl', 'Opus', 'HelveticaNeue UltraLigExt', 'Myriad Pro Light It', 'HelveticaNeue UltraLigExtObl', 'Opus Chords Sans', 'HO Futura HeavyOblique', 'Opus Japanese Chords', 'L Frutiger Light', 'VT100', 'L Futura Light', 'Helsinki', 'LO Futura LightOblique', 'Helsinki Metronome', 'Myriad Pro Black', 'New York', 'O Futura BookOblique', 'R Frutiger Roman', 'Reprise', 'TradeGothic', 'Warnock Pro Bold Caption', 'Univers 45 Light', 'Warnock Pro', 'XBO Futura ExtraBoldOblique', 'Univers 45 LightOblique', 'Liberation Mono', 'Univers 55 Oblique', 'UC LCD', 'Univers 57 Condensed', 'Warnock Pro Bold', 'Univers ExtraBlack', 'Warnock Pro Light Ital Subhead', 'Univers LightUltraCondensed', 'Matrix Ticker', 'Univers UltraCondensed', 'Fang Song'];
            const availableFonts = [];
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            fontList.forEach(font => {
                ctx.font = `12px ${font}`;
                if (ctx.font.indexOf(font) !== -1) {
                    availableFonts.push(font);
                }
            });
            return availableFonts;
        })(),

        // 8 - Use of Adblock
        adBlock: (() => {
            const adBlockEnabled = document.createElement('div');
            adBlockEnabled.innerHTML = '&nbsp;';
            adBlockEnabled.className = 'adsbox';
            document.body.appendChild(adBlockEnabled);
            const isAdBlockEnabled = adBlockEnabled.offsetHeight === 0;
            document.body.removeChild(adBlockEnabled);
            return isAdBlockEnabled;
        })(),

        // 9 - Do Not Track
        doNotTrack: navigator.doNotTrack || navigator.msDoNotTrack || 'Not supported',

        // 10 - Navigator properties
        navigatorProperties: Object.keys(navigator).length,

        // 11 - BuildID
        buildID: navigator.buildID || 'Not supported',

        // 12 - Product
        product: navigator.product || 'Not supported',

        // 13 - Product sub
        productSub: navigator.productSub || 'Not supported',

        // 14 - Vendor
        vendor: navigator.vendor || 'Not supported',

        // 15 - Vendor sub
        vendorSub: navigator.vendorSub || 'Not supported',

        // 16 - Hardware concurrency
        hardwareConcurrency: navigator.hardwareConcurrency || 'Not supported',

        // 17 - Java enabled
        javaEnabled: navigator.javaEnabled(),

        // 18 - Device memory
        deviceMemory: navigator.deviceMemory || 'Not supported',

        // 19 - List of plugins
        plugins: Array.from(navigator.plugins).map(plugin => plugin.name) || 'Not supported',

        // 20 - Screen width
        screenWidth: screen.width,

        // 21 - Screen height
        screenHeight: screen.height,

        // 22 - Screen depth
        screenDepth: screen.colorDepth,

        // 23 - Screen available top
        screenAvailableTop: screen.availTop !== undefined ? screen.availTop : 'Not supported',

        // 24 - Screen available left
        screenAvailableLeft: screen.availLeft !== undefined ? screen.availLeft : 'Not supported',

        // 25 - Screen available height
        screenAvailableHeight: screen.availHeight !== undefined ? screen.availHeight : 'Not supported',

        // 26 - Screen available width
        screenAvailableWidth: screen.availWidth !== undefined ? screen.availWidth : 'Not supported',

        // 27 - Screen left
        screenLeft: window.screenLeft !== undefined ? window.screenLeft : 'Not supported',

        // 28 - Screen top
        screenTop: window.screenTop !== undefined ? window.screenTop : 'Not supported',

        // 29 - Permissions
        permissions: (async () => {
            if (navigator.permissions) {
                const permissions = {};
                const types = ['geolocation', 'notifications', 'camera', 'microphone', 'midi', 'payment-handler', 'push'];
                for (const type of types) {
                    try {
                        const result = await navigator.permissions.query({ name: type });
                        permissions[type] = result.state;
                    } catch (e) {
                        permissions[type] = 'Not supported';
                    }
                }
                return permissions;
            } else {
                return 'Permissions API not supported';
            }
        })(),

        // 30 - WebGL Vendor
        webGLVendor: (() => {
            try {
                const canvas = document.createElement('canvas');
                const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
                const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');
                return debugInfo ? gl.getParameter(debugInfo.UNMASKED_VENDOR_WEBGL) : 'Not supported';
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 31 - WebGL Renderer
        webGLRenderer: (() => {
            try {
                const canvas = document.createElement('canvas');
                const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
                const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');
                return debugInfo ? gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL) : 'Not supported';
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 32 - WebGL Data
        webGLData: (() => {
            try {
                const canvas = document.createElement('canvas');
                const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
                return gl ? 'WebGL supported' : 'Not supported';
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 33 - WebGL Parameters
        webGLParameters: (() => {
            try {
                const canvas = document.createElement('canvas');
                const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
                const parameters = [
                    'ARRAY_BUFFER_BINDING',
                    'ELEMENT_ARRAY_BUFFER_BINDING',
                    'CURRENT_PROGRAM',
                    'CURRENT_VERTEX_ATTRIB',
                    'DEPTH_BITS',
                    'STENCIL_BITS'
                ];
                const values = {};
                parameters.forEach(param => {
                    values[param] = gl.getParameter(gl[param]);
                });
                return values;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 34 - Use of local storage
        localStorage: !!window.localStorage ? 'Supported' : 'Not supported',

        // 35 - Use of session storage
        sessionStorage: !!window.sessionStorage ? 'Supported' : 'Not supported',

        // 36 - Use of IndexedDB
        indexedDB: !!window.indexedDB ? 'Supported' : 'Not supported',

        // 37 - Audio formats
        audioFormats: (() => {
            const formats = [
                'audio/aac', 'audio/flac', 'audio/mpeg', 'audio/ogg; codecs="flac"',
                'audio/ogg; codecs="vorbis"', 'audio/ogg; codecs="opus"',
                'audio/wav; codecs="1"', 'audio/webm; codecs="vorbis"',
                'audio/webm; codecs="opus"', 'audio/mp4; codecs="mp4a_40_2"'
            ];
            const canPlay = {};
            formats.forEach(format => {
                canPlay[format] = document.createElement('audio').canPlayType(format);
            });
            return canPlay;
        })(),

        // 38 - Audio context
        audioContext: (() => {
            try {
                const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                return {
                    sampleRate: audioCtx.sampleRate,
                    state: audioCtx.state
                };
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 39 - Frequency analyser
        frequencyAnalyser: (() => {
            try {
                const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                const analyser = audioCtx.createAnalyser();
                return {
                    channelCount: analyser.channelCount,
                    channelCountMode: analyser.channelCountMode,
                    channelInterpretation: analyser.channelInterpretation,
                    fftSize: analyser.fftSize,
                    frequencyBinCount: analyser.frequencyBinCount,
                    maxDecibels: analyser.maxDecibels,
                    minDecibels: analyser.minDecibels,
                    numberOfInputs: analyser.numberOfInputs,
                    numberOfOutputs: analyser.numberOfOutputs,
                    smoothingTimeConstant: analyser.smoothingTimeConstant
                };
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 40 - Audio data
        audioData: (() => {
            try {
                const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                return audioCtx;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 41 - Video formats
        videoFormats: (() => {
            const formats = [
                'video/mp4; codecs="flac"', 'video/ogg; codecs="theora"',
                'video/ogg; codecs="opus"', 'video/webm; codecs="vp9, opus"',
                'video/webm; codecs="vp8, vorbis"'
            ];
            const canPlay = {};
            formats.forEach(format => {
                canPlay[format] = document.createElement('video').canPlayType(format);
            });
            return canPlay;
        })(),

        // 42 - Media devices
        mediaDevices: (async () => {
            try {
                const devices = await navigator.mediaDevices.enumerateDevices();
                return devices.map(device => `${device.kind}: ${device.label || 'Unnamed'}`);
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 43 - Accelerometer
        accelerometer: (() => {
            try {
                if (window.DeviceMotionEvent) {
                    return 'Supported';
                }
            } catch (e) {
                return 'Not supported';
            }
            return 'Not supported';
        })(),

        // 44 - Gyroscope
        gyroscope: (() => {
            try {
                if (window.DeviceOrientationEvent) {
                    return 'Supported';
                }
            } catch (e) {
                return 'Not supported';
            }
            return 'Not supported';
        })(),

        // 45 - Proximity sensor
        proximitySensor: 'Not supported', // No direct API for proximity sensor in most browsers

        // 46 - Keyboard layout
        keyboardLayout: 'Not supported', // No direct API for keyboard layout in most browsers

        // 47 - Battery
        battery: (async () => {
            try {
                if (navigator.getBattery) {
                    const battery = await navigator.getBattery();
                    return {
                        charging: battery.charging,
                        level: battery.level,
                        chargingTime: battery.chargingTime,
                        dischargingTime: battery.dischargingTime
                    };
                }
            } catch (e) {
                return 'Not supported';
            }
            return 'Not supported';
        })(),

        // 48 - Connection
        connection: navigator.connection ? {
            effectiveType: navigator.connection.effectiveType,
            downlink: navigator.connection.downlink,
            rtt: navigator.connection.rtt
        } : 'Not supported',

        // 49 - Key
        key: 'No value',

        // 50 - Location bar
        locationBar: window.outerWidth ? 'Supported' : 'Not supported',

        // 51 - Menu bar
        menuBar: window.outerWidth ? 'Supported' : 'Not supported',

        // 52 - Personal bar
        personalBar: window.outerWidth ? 'Supported' : 'Not supported',

        // 53 - Status bar
        statusBar: window.outerWidth ? 'Supported' : 'Not supported',

        // 54 - Tool bar
        toolBar: window.outerWidth ? 'Supported' : 'Not supported',

        // 55 - Result state
        resultState: 'No value',

        // 56 - List of fonts (Flash)
        fontsFlash: 'Flash not detected',

        // 57 - Screen resolution (Flash)
        screenResolutionFlash: 'Flash not detected',

        // 58 - Language (Flash)
        languageFlash: 'Flash not detected',

        // 59 - Platform (Flash)
        platformFlash: 'Flash not detected',

        // ===== Tests supplémentaires =====

        // 60 - User-Agent Client Hints (nouveau standard, remplace progressivement l'UA)
        userAgentData: navigator.userAgentData ? {
            brands: navigator.userAgentData.brands,
            mobile: navigator.userAgentData.mobile,
            platform: navigator.userAgentData.platform
        } : 'Not supported',

        // 61 - Client Hints haute entropie (architecture CPU, version complète, modèle...)
        highEntropyHints: (async () => {
            try {
                if (navigator.userAgentData && navigator.userAgentData.getHighEntropyValues) {
                    return await navigator.userAgentData.getHighEntropyValues(
                        ['architecture', 'bitness', 'model', 'platformVersion', 'uaFullVersion', 'fullVersionList']
                    );
                }
            } catch (e) {}
            return 'Not supported';
        })(),

        // 62 - WebGL2 : version, extensions complètes et paramètres GPU
        webGL2Info: (() => {
            try {
                const canvas = document.createElement('canvas');
                const gl = canvas.getContext('webgl2');
                if (!gl) return 'Not supported';
                const params = {};
                ['MAX_TEXTURE_SIZE', 'MAX_RENDERBUFFER_SIZE', 'MAX_VERTEX_ATTRIBS', 'MAX_VARYING_VECTORS',
                 'MAX_VERTEX_UNIFORM_VECTORS', 'MAX_FRAGMENT_UNIFORM_VECTORS', 'MAX_TEXTURE_IMAGE_UNITS',
                 'MAX_COMBINED_TEXTURE_IMAGE_UNITS', 'ALIASED_LINE_WIDTH_RANGE', 'ALIASED_POINT_SIZE_RANGE',
                 'MAX_VIEWPORT_DIMS', 'SAMPLES'].forEach(p => {
                    try { params[p] = gl.getParameter(gl[p]); } catch (e) { params[p] = 'Error'; }
                });
                return {
                    version: gl.getParameter(gl.VERSION),
                    shadingLanguageVersion: gl.getParameter(gl.SHADING_LANGUAGE_VERSION),
                    extensions: gl.getSupportedExtensions(),
                    parameters: params
                };
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 63 - WebGL image hash : rendu d'un triangle + hash de l'image (varie selon le GPU/driver)
        webGLImageHash: (() => {
            try {
                const canvas = document.createElement('canvas');
                canvas.width = 100; canvas.height = 100;
                const gl = canvas.getContext('webgl');
                const vs = 'attribute vec2 p;void main(){gl_Position=vec4(p,0.0,1.0);}';
                const fs = 'precision mediump float;void main(){gl_FragColor=vec4(0.73,0.88,0.10,1.0);}';
                const compile = (type, src) => {
                    const s = gl.createShader(type);
                    gl.shaderSource(s, src);
                    gl.compileShader(s);
                    return s;
                };
                const prog = gl.createProgram();
                gl.attachShader(prog, compile(gl.VERTEX_SHADER, vs));
                gl.attachShader(prog, compile(gl.FRAGMENT_SHADER, fs));
                gl.linkProgram(prog);
                gl.useProgram(prog);
                const buf = gl.createBuffer();
                gl.bindBuffer(gl.ARRAY_BUFFER, buf);
                gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([-0.5, 0.5, -0.5, -0.5, 0.5, 0.5, 0.5, -0.5]), gl.STATIC_DRAW);
                const loc = gl.getAttribLocation(prog, 'p');
                gl.enableVertexAttribArray(loc);
                gl.vertexAttribPointer(loc, 2, gl.FLOAT, false, 0, 0);
                gl.clearColor(0.2, 0.4, 0.6, 1.0);
                gl.clear(gl.COLOR_BUFFER_BIT);
                gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
                const url = canvas.toDataURL();
                let hash = 0;
                for (let i = 0; i < url.length; i++) {
                    hash = ((hash << 5) - hash + url.charCodeAt(i)) | 0;
                }
                return hash;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 64 - Canvas emoji : le rendu des emoji varie selon l'OS (moteur de rendu système)
        emojiFingerprint: (() => {
            try {
                const canvas = document.createElement('canvas');
                canvas.width = 140; canvas.height = 40;
                const ctx = canvas.getContext('2d');
                ctx.textBaseline = 'alphabetic';
                ctx.font = '30px Arial';
                ctx.fillStyle = '#f60';
                ctx.fillRect(0, 0, 140, 40);
                ctx.fillStyle = '#069';
                ctx.fillText('😀克隆🇫🇷👍', 2, 30);
                const url = canvas.toDataURL();
                let hash = 0;
                for (let i = 0; i < url.length; i++) {
                    hash = ((hash << 5) - hash + url.charCodeAt(i)) | 0;
                }
                return hash;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 65 - Audio fingerprint classique : rendu OfflineAudioContext (oscillateur + compresseur) et somme du buffer
        audioFingerprint: (async () => {
            try {
                const ctx = new OfflineAudioContext(1, 44100, 44100);
                const oscillator = ctx.createOscillator();
                oscillator.type = 'triangle';
                oscillator.frequency.value = 10000;
                const compressor = ctx.createDynamicsCompressor();
                compressor.threshold.value = -50;
                compressor.knee.value = 40;
                compressor.ratio.value = 12;
                compressor.attack.value = 0;
                compressor.release.value = 0.25;
                oscillator.connect(compressor);
                compressor.connect(ctx.destination);
                oscillator.start(0);
                const buffer = await ctx.startRendering();
                const data = buffer.getChannelData(0);
                let sum = 0;
                for (let i = 4500; i < 5000; i++) sum += Math.abs(data[i]);
                return sum;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 66 - ClientRects : dimensions au pixel près d'éléments HTML rendus (police + moteur de rendu)
        clientRects: (() => {
            try {
                const container = document.createElement('div');
                container.style.cssText = 'position:absolute;left:-9999px;font-size:36px;';
                container.innerHTML = '<span style="font-family:serif">http://nyan.cat</span><b style="font-size:8px">iii</b>';
                document.body.appendChild(container);
                const range = document.createRange();
                range.selectNode(container);
                const rects = Array.from(range.getClientRects()).map(r => `${r.width.toFixed(2)}x${r.height.toFixed(2)}`).join(', ');
                document.body.removeChild(container);
                return rects;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 67 - Détection des polices par mesure de largeur (technique plus fiable que ctx.font)
        fontsByMeasure: (() => {
            try {
                const candidates = ['Arial', 'Verdana', 'Times New Roman', 'Courier New', 'Georgia', 'Comic Sans MS',
                    'Impact', 'Trebuchet MS', 'Consolas', 'Segoe UI', 'Roboto', 'Helvetica Neue', 'Calibri',
                    'Cambria', 'Ubuntu', 'DejaVu Sans', 'Liberation Sans', 'Noto Sans', 'Menlo', 'Monaco'];
                const base = ['monospace', 'sans-serif', 'serif'];
                const span = document.createElement('span');
                span.style.cssText = 'position:absolute;left:-9999px;white-space:nowrap;';
                span.textContent = 'mmmmmmmmmmlli Ww@#';
                document.body.appendChild(span);
                const measure = (font) => {
                    span.style.font = `72px ${font}`;
                    return span.offsetWidth + '/' + span.offsetHeight;
                };
                const baseSizes = {};
                base.forEach(b => baseSizes[b] = measure(b));
                const detected = candidates.filter(f => base.some(b => measure(`${f},${b}`) !== baseSizes[b]));
                document.body.removeChild(span);
                return detected;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 68 - Voix de synthèse vocale (très révélateur de l'OS et des langues installées)
        speechVoices: (() => {
            try {
                return speechSynthesis.getVoices().map(v => `${v.name} (${v.lang})${v.localService ? '' : ' [remote]'}`);
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 69 - WebRTC : fuite des IP locales (mDNS désactivé) et candidats ICE
        webrtcLocalIps: (() => {
            return new Promise((resolve) => {
                try {
                    const pc = new RTCPeerConnection({ iceServers: [] });
                    const ips = new Set();
                    pc.createDataChannel('test');
                    pc.onicecandidate = (e) => {
                        if (!e.candidate) {
                            resolve(Array.from(ips));
                            return;
                        }
                        const match = e.candidate.candidate.match(/([0-9]{1,3}(\.[0-9]{1,3}){3}|[a-f0-9]{4}(:[a-f0-9]{4}){2,7})/i);
                        if (match) ips.add(match[1]);
                    };
                    pc.createOffer().then(offer => pc.setLocalDescription(offer));
                    setTimeout(() => resolve(Array.from(ips)), 2000);
                } catch (e) {
                    resolve('Not supported');
                }
            });
        })(),

        // 70 - Math fingerprint : la précision des fonctions mathématiques varie selon le moteur JS
        mathFingerprint: (() => {
            const ops = {
                tan: Math.tan(-1e300),
                sin: Math.sin(1e300),
                cos: Math.cos(1e300),
                acos: Math.acos(0.123),
                atan: Math.atan(2),
                log: Math.log(1e300),
                exp: Math.exp(700),
                sinh: Math.sinh(1),
                pow: Math.pow(Math.PI, -100)
            };
            return Object.values(ops).join(',');
        })(),

        // 71 - Format des messages d'erreur JS (spécifique au navigateur)
        errorFingerprint: (() => {
            try {
                null.x;
            } catch (e) {
                return `${e.name}: ${e.message}`;
            }
            return 'No error';
        })(),

        // 72 - Format de la stack trace (Chrome / Firefox / Safari ont des formats très différents)
        stackTrace: (() => {
            try {
                throw new Error('fingerprint');
            } catch (e) {
                return {
                    lines: e.stack.split('\n').length,
                    format: e.stack.split('\n').slice(0, 3)
                };
            }
        })(),

        // 73 - Nombre de propriétés globales du window (très variable selon navigateur/version/extensions)
        globalProperties: Object.getOwnPropertyNames(window).length,

        // 74 - Prototype de Navigator : liste des propriétés/méthodes disponibles
        navigatorPrototype: Object.getOwnPropertyNames(Navigator.prototype).join(','),

        // 75 - Détection des API modernes disponibles (révèle navigateur + flags expérimentaux)
        apisAvailable: {
            webgpu: !!navigator.gpu,
            webserial: !!navigator.serial,
            webusb: !!navigator.usb,
            webhid: !!navigator.hid,
            webnfc: 'NDEFReader' in window,
            bluetooth: !!navigator.bluetooth,
            webxr: !!navigator.xr,
            wakeLock: !!navigator.wakeLock,
            webauthn: !!window.PublicKeyCredential,
            paymentRequest: !!window.PaymentRequest,
            idleDetector: !!window.IdleDetector,
            computePressure: !!window.PressureObserver,
            fileSystemAccess: !!window.showOpenFilePicker,
            eyeDropper: !!window.EyeDropper,
            contactPicker: !!window.ContactsManager,
            virtualKeyboard: !!navigator.virtualKeyboard,
            devicePosture: !!navigator.devicePosture,
            browsingTopics: typeof document.browsingTopics === 'function',
            sharedArrayBuffer: typeof SharedArrayBuffer !== 'undefined'
        },

        // 76 - WebGPU : infos du GPU (vendor, architecture, device)
        webgpuInfo: (async () => {
            try {
                if (navigator.gpu) {
                    const adapter = await navigator.gpu.requestAdapter();
                    if (!adapter) return 'No adapter';
                    if (adapter.info && Object.keys(adapter.info).length) return adapter.info;
                    if (adapter.requestAdapterInfo) return await adapter.requestAdapterInfo();
                    return 'WebGPU supported (no info)';
                }
            } catch (e) {}
            return 'Not supported';
        })(),

        // 77 - Écran étendu : DPR, orientation, multi-écrans, dimensions de la fenêtre
        screenExtra: {
            devicePixelRatio: window.devicePixelRatio,
            orientation: screen.orientation ? { type: screen.orientation.type, angle: screen.orientation.angle } : 'Not supported',
            isExtended: screen.isExtended !== undefined ? screen.isExtended : 'Not supported',
            inner: `${window.innerWidth}x${window.innerHeight}`,
            outer: `${window.outerWidth}x${window.outerHeight}`
        },

        // 78 - Support tactile
        touchSupport: {
            maxTouchPoints: navigator.maxTouchPoints,
            touchEvent: 'ontouchstart' in window,
            touchStart: typeof TouchEvent !== 'undefined',
            pointerCoarse: window.matchMedia('(pointer: coarse)').matches
        },

        // 79 - Media queries CSS : préférences système (thème, animations, contraste, HDR...)
        cssMediaQueries: (() => {
            const queries = {
                pointerFine: '(pointer: fine)',
                hover: '(hover: hover)',
                colorGamutP3: '(color-gamut: p3)',
                colorGamutRec2020: '(color-gamut: rec2020)',
                darkMode: '(prefers-color-scheme: dark)',
                reducedMotion: '(prefers-reduced-motion: reduce)',
                moreContrast: '(prefers-contrast: more)',
                lessContrast: '(prefers-contrast: less)',
                forcedColors: '(forced-colors: active)',
                hdr: '(dynamic-range: high)',
                standalone: '(display-mode: standalone)',
                invertedColors: '(inverted-colors: inverted)',
                monochrome: '(monochrome: 0)'
            };
            const results = {};
            for (const [name, q] of Object.entries(queries)) {
                try { results[name] = window.matchMedia(q).matches; } catch (e) { results[name] = 'Error'; }
            }
            return results;
        })(),

        // 80 - Support des fonctionnalités CSS récentes (révèle la version du moteur)
        cssFeatures: (() => {
            try {
                const results = {};
                ['backdrop-filter: blur(1px)', 'clip-path: circle(1px)', 'display: grid', 'position: sticky',
                 'aspect-ratio: 1/1', 'gap: 1px', 'overflow: clip', 'text-wrap: balance'].forEach(f => {
                    results[f] = CSS.supports(f);
                });
                results['selector :has()'] = CSS.supports('selector(:has(a))');
                results['container queries'] = CSS.supports('container-type: inline-size');
                results['subgrid'] = CSS.supports('grid-template-rows: subgrid');
                return results;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 81 - Styles par défaut des éléments système (police des boutons/inputs selon OS)
        systemStyles: (() => {
            try {
                const button = document.createElement('button');
                const input = document.createElement('input');
                document.body.appendChild(button);
                document.body.appendChild(input);
                const bs = getComputedStyle(button);
                const is = getComputedStyle(input);
                const res = {
                    buttonFont: `${bs.fontFamily} ${bs.fontSize}`,
                    inputFont: `${is.fontFamily} ${is.fontSize}`,
                    accentColor: bs.accentColor || 'Not supported'
                };
                document.body.removeChild(button);
                document.body.removeChild(input);
                return res;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 82 - Liste des MimeTypes supportés
        mimeTypes: Array.from(navigator.mimeTypes || []).map(m => m.type),

        // 83 - PDF viewer intégré
        pdfViewerEnabled: navigator.pdfViewerEnabled !== undefined ? navigator.pdfViewerEnabled : 'Not supported',

        // 84 - webdriver : détection d'automatisation (Selenium, Puppeteer...)
        webdriver: navigator.webdriver !== undefined ? navigator.webdriver : 'Not supported',

        // 85 - Permission notifications
        notificationPermission: ('Notification' in window) ? Notification.permission : 'Not supported',

        // 86 - Quota de stockage : révèle approximativement la taille du disque !
        storageEstimate: (async () => {
            try {
                if (navigator.storage && navigator.storage.estimate) {
                    const { quota, usage } = await navigator.storage.estimate();
                    return { quotaGo: (quota / 1e9).toFixed(2), usageMo: (usage / 1e6).toFixed(2) };
                }
            } catch (e) {}
            return 'Not supported';
        })(),

        // 87 - Mémoire JS heap (Chrome uniquement)
        performanceMemory: performance.memory ? {
            jsHeapSizeLimit: performance.memory.jsHeapSizeLimit,
            totalJSHeapSize: performance.memory.totalJSHeapSize,
            usedJSHeapSize: performance.memory.usedJSHeapSize
        } : 'Not supported',

        // 88 - Résolution des timers (précision variable anti-fingerprinting / rAF period)
        timerResolution: (() => {
            return new Promise(resolve => {
                let settled = false;
                const deltas = [];
                let last = performance.now();
                const finish = () => {
                    if (settled) return;
                    settled = true;
                    if (!deltas.length) {
                        resolve('rAF indisponible');
                        return;
                    }
                    const positive = deltas.filter(d => d > 0);
                    resolve({
                        rafPeriod: +(positive.reduce((a, b) => a + b, 0) / positive.length).toFixed(2),
                        minDelta: +Math.min(...positive).toFixed(4)
                    });
                };
                const tick = () => {
                    const now = performance.now();
                    deltas.push(now - last);
                    last = now;
                    if (deltas.length < 50 && !settled) {
                        requestAnimationFrame(tick);
                    } else {
                        finish();
                    }
                };
                requestAnimationFrame(tick);
                setTimeout(finish, 2000);
            });
        })(),

        // 89 - UA dans un Web Worker : permet de détecter un spoof d'User-Agent incohérent
        workerUserAgent: (() => {
            return new Promise((resolve) => {
                try {
                    const code = "postMessage({ua: navigator.userAgent, platform: navigator.platform, hardwareConcurrency: navigator.hardwareConcurrency})";
                    const worker = new Worker(URL.createObjectURL(new Blob([code], { type: 'application/javascript' })));
                    worker.onmessage = (e) => {
                        resolve({
                            sameUA: e.data.ua === navigator.userAgent,
                            workerUA: e.data.ua,
                            workerPlatform: e.data.platform
                        });
                    };
                    worker.onerror = () => resolve('Not supported');
                } catch (e) {
                    resolve('Not supported');
                }
            });
        })(),

        // 90 - Intl API : formats de nombres/dates/collation spécifiques à la locale du système
        intlFingerprint: (() => {
            try {
                return {
                    number: new Intl.NumberFormat().format(123456.789),
                    currency: new Intl.NumberFormat(undefined, { style: 'currency', currency: 'EUR' }).format(1234.5),
                    dateParts: new Intl.DateTimeFormat().formatToParts(new Date()).map(p => p.type).join('|'),
                    collator: JSON.stringify(new Intl.Collator().resolvedOptions()),
                    timezonesCount: (typeof Intl.supportedValuesOf === 'function') ? Intl.supportedValuesOf('timeZone').length : 'Not supported'
                };
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 91 - Disposition clavier réelle (Chrome, nécessite focus sur la page)
        keyboardMap: (async () => {
            try {
                if (navigator.keyboard && navigator.keyboard.getLayoutMap) {
                    const map = await navigator.keyboard.getLayoutMap();
                    return ['KeyQ', 'KeyW', 'KeyY', 'Semicolon', 'Backquote'].map(k => `${k}:${map.get(k)}`).join(' ');
                }
            } catch (e) {}
            return 'Not supported';
        })(),

        // 92 - Manettes de jeu connectées
        gamepads: (() => {
            try {
                const pads = Array.from(navigator.getGamepads()).filter(Boolean).map(g => g.id);
                return pads.length ? pads : 'No gamepad';
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 93 - WebXR : support casque VR/AR
        xrSupport: (async () => {
            try {
                if (navigator.xr) {
                    return {
                        immersiveVr: await navigator.xr.isSessionSupported('immersive-vr'),
                        immersiveAr: await navigator.xr.isSessionSupported('immersive-ar')
                    };
                }
            } catch (e) {}
            return 'Not supported';
        })(),

        // 94 - Media Capabilities : capacités de décodage matériel (4K H264, AV1, VP9...)
        mediaCapabilities: (async () => {
            try {
                const configs = [
                    { type: 'file', video: { contentType: 'video/mp4; codecs="avc1.640028"', width: 3840, height: 2160, bitrate: 20000000 } },
                    { type: 'file', video: { contentType: 'video/webm; codecs="av01.0.08M.08"', width: 1920, height: 1080, bitrate: 8000000 } },
                    { type: 'file', video: { contentType: 'video/webm; codecs="vp09.00.10.08"', width: 1920, height: 1080, bitrate: 5000000 } }
                ];
                const results = {};
                for (const cfg of configs) {
                    const r = await navigator.mediaCapabilities.decodingInfo(cfg);
                    results[cfg.video.contentType] = { supported: r.supported, smooth: r.smooth, powerEfficient: r.powerEfficient };
                }
                return results;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 95 - Support DRM (Widevine, PlayReady, FairPlay)
        drmSupport: (async () => {
            const systems = ['com.widevine.alpha', 'com.microsoft.playready', 'com.apple.fps.1_0', 'org.w3.clearkey'];
            const results = {};
            for (const system of systems) {
                try {
                    await navigator.requestMediaKeySystemAccess(system, [{
                        initDataTypes: ['cenc'],
                        videoCapabilities: [{ contentType: 'video/mp4; codecs="avc1.640028"' }]
                    }]);
                    results[system] = 'Supported';
                } catch (e) {
                    results[system] = 'Not supported';
                }
            }
            return results;
        })(),

        // 96 - API Crypto
        cryptoSupport: {
            subtle: !!crypto.subtle,
            randomUUID: typeof crypto.randomUUID === 'function',
            getRandomValues: typeof crypto.getRandomValues === 'function'
        },

        // 97 - Capteurs génériques (API Sensors)
        sensorsApi: (() => {
            const sensors = ['Accelerometer', 'Gyroscope', 'Magnetometer', 'AmbientLightSensor', 'AbsoluteOrientationSensor', 'GravitySensor'];
            const res = {};
            sensors.forEach(s => res[s] = s in window);
            return res;
        })(),

        // 98 - Connection : infos supplémentaires (type réseau, économie de données)
        connectionExtra: navigator.connection ? {
            type: navigator.connection.type !== undefined ? navigator.connection.type : 'Not supported',
            saveData: navigator.connection.saveData !== undefined ? navigator.connection.saveData : 'Not supported'
        } : 'Not supported',

        // 99 - Timezone : offsets janvier/juillet (détection DST et fuseau réel malgré le spoof)
        timezoneOffsets: (() => {
            const year = new Date().getFullYear();
            const jan = new Date(year, 0, 1).getTimezoneOffset();
            const jul = new Date(year, 6, 1).getTimezoneOffset();
            return { january: jan, july: jul, dst: jan !== jul };
        })(),

        // 100 - Storage Access / Privacy Sandbox (Topics, Protected Audience, attribution)
        privacyApis: ({
            hasStorageAccess: typeof document.hasStorageAccess === 'function',
            storageBuckets: !!(navigator.storage && navigator.storage.getBucket),
            protectedAudience: !!navigator.protectedAudience,
            attributionReporting: typeof window.attributionReporting !== 'undefined',
            privateStateTokens: typeof document.hasPrivateToken === 'function'
        }),

        // 101 - measureText : métriques de rendu du texte (moteur de polices)
        measureText: (() => {
            try {
                const canvas = document.createElement('canvas');
                const ctx = canvas.getContext('2d');
                ctx.font = '16px Arial';
                const m1 = ctx.measureText('Hello World! https://example.com 123');
                ctx.font = 'italic 700 20px serif';
                const m2 = ctx.measureText('Ag 🇫🇷 Éé');
                return [
                    m1.width, m1.actualBoundingBoxAscent, m1.actualBoundingBoxDescent,
                    m2.width, m2.actualBoundingBoxLeft, m2.actualBoundingBoxRight
                ].join(',');
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 102 - Sortie audio : nombre de canaux matériels (carte son / casque)
        audioOutput: (() => {
            try {
                const ctx = new (window.AudioContext || window.webkitAudioContext)();
                const dest = ctx.destination;
                return {
                    channelCount: dest.channelCount,
                    maxChannelCount: dest.maxChannelCount,
                    numberOfInputs: dest.numberOfInputs,
                    numberOfOutputs: dest.numberOfOutputs
                };
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 103 - WebAssembly : features du moteur (SIMD)
        webassemblyFeatures: {
            supported: typeof WebAssembly === 'object',
            simd: (() => {
                try {
                    return WebAssembly.validate(new Uint8Array([0, 97, 115, 109, 1, 0, 0, 0, 1, 5, 1, 96, 0, 1, 123, 3, 2, 1, 0, 10, 10, 1, 8, 0, 65, 0, 253, 15, 253, 98, 11]));
                } catch (e) {
                    return 'Error';
                }
            })()
        },

        // 104 - Largeur de scrollbar (dépend OS / navigateur / mobile)
        scrollbarWidth: (() => {
            try {
                const outer = document.createElement('div');
                outer.style.cssText = 'position:absolute;left:-9999px;width:100px;height:100px;overflow:scroll;';
                document.body.appendChild(outer);
                const width = outer.offsetWidth - outer.clientWidth;
                document.body.removeChild(outer);
                return width;
            } catch (e) {
                return 'Not supported';
            }
        })(),

        // 105 - Détection du moteur / navigateur (indices classiques)
        engineDetection: {
            chromium: !!window.chrome,
            firefox: typeof InstallTrigger !== 'undefined',
            opera: (!!window.opr && !!window.opr.addons) || !!window.opera,
            edgeChromium: navigator.userAgent.indexOf('Edg/') > -1,
            ieLegacy: !!document.documentMode
        },

        // 106 - Permissions Policy : features autorisées pour ce document (Chrome)
        featurePolicy: (() => {
            try {
                if (document.featurePolicy) return document.featurePolicy.allowedFeatures();
                if (document.permissionsPolicy) return document.permissionsPolicy.allowedFeatures();
            } catch (e) {}
            return 'Not supported';
        })(),

        // 107 - Cookie Store API + persistance du stockage
        storageApis: (async () => {
            const res = { cookieStore: !!window.cookieStore };
            try {
                res.persisted = await navigator.storage.persisted();
            } catch (e) {
                res.persisted = 'Not supported';
            }
            return res;
        })(),

        // 108 - PWA installées liées au site
        relatedApps: (async () => {
            try {
                if (navigator.getInstalledRelatedApps) {
                    const apps = await navigator.getInstalledRelatedApps();
                    return apps.length ? apps.map(a => a.id || a.platform) : 'No related app';
                }
            } catch (e) {}
            return 'Not supported';
        })(),

        // 109 - Disponibilité Bluetooth
        bluetoothAvailability: (async () => {
            try {
                if (navigator.bluetooth && navigator.bluetooth.getAvailability) {
                    return (await navigator.bluetooth.getAvailability()) ? 'Available' : 'No Bluetooth adapter';
                }
            } catch (e) {}
            return 'Not supported';
        })(),

        // 110 - Outils anti-pistage : les trackers connus sont-ils bloqués ? (uBlock, Brave...)
        antiTracking: (async () => {
            const targets = {
                'google-analytics': 'https://www.google-analytics.com/analytics.js',
                'doubleclick': 'https://securepubads.g.doubleclick.net/pagead/id.js',
                'facebook-pixel': 'https://connect.facebook.net/en_US/fbevents.js'
            };
            const results = {};
            for (const [name, url] of Object.entries(targets)) {
                const ctrl = new AbortController();
                const timer = setTimeout(() => ctrl.abort(), 3000);
                try {
                    await fetch(url, { mode: 'no-cors', cache: 'no-store', signal: ctrl.signal });
                    results[name] = 'accessible (non bloqué)';
                } catch (e) {
                    results[name] = 'bloqué';
                }
                clearTimeout(timer);
            }
            return results;
        })(),

        // 111 - Navigation : historique, type de navigation, encodage
        navigationInfo: {
            historyLength: history.length,
            navigationType: (() => {
                try {
                    return performance.getEntriesByType('navigation')[0].type;
                } catch (e) {
                    return 'Not supported';
                }
            })(),
            visibilityState: document.visibilityState,
            characterSet: document.characterSet
        },

        // 112 - Liste complète des langues préférées
        languages: Array.from(navigator.languages || [navigator.language]),

        // 113 - Vibration (mobile)
        vibrationSupport: ('vibrate' in navigator) ? 'Supported' : 'Not supported',

        // 114 - Permissions étendues (API récentes du navigateur)
        permissionsExtended: (async () => {
            if (!navigator.permissions) return 'Not supported';
            const names = ['clipboard-read', 'clipboard-write', 'midi', 'persistent-storage',
                'background-sync', 'screen-wake-lock', 'accelerometer', 'magnetometer',
                'ambient-light-sensor', 'local-fonts', 'window-management', 'storage-access'];
            const out = {};
            for (const name of names) {
                try {
                    out[name] = (await navigator.permissions.query({ name })).state;
                } catch (e) {
                    out[name] = 'Not supported';
                }
            }
            return out;
        })()
    };

    window.__fpRaw = browserInfo;
})();


</script>

<script>
    (function() {
        async function safeStringify(value) {
            try {
                const s = JSON.stringify(value, null, 2);
                return s === undefined ? String(value) : s;
            } catch (e) {
                try { return String(value); } catch (e2) { return '[Unserializable]'; }
            }
        }

        async function renderFingerprint() {
            const status = document.getElementById('fp-status');
            const container = document.getElementById('fp-results');
            if (!status || !container) return;
            status.textContent = '⏳ Collecte des données...';
            container.innerHTML = '';

            const data = window.__fpRaw;
            if (!data) {
                status.textContent = '❌ Le script de collecte n\'a pas pu s\'exécuter sur ce navigateur (voir console).';
                return;
            }
            const rows = [];
            for (const [key, value] of Object.entries(data)) {
                if (key === 'jsAttributes') {
                    for (const [k2, v2] of Object.entries(value)) rows.push([k2, v2]);
                } else {
                    rows.push([key, value]);
                }
            }

            const withTimeout = (promise, ms, label) => Promise.race([
                promise,
                new Promise(res => setTimeout(() => res('⏱️ timeout (' + label + ')'), ms))
            ]);

            const total = rows.length;
            const full = {};
            let done = 0;

            // Lancement de tous les tests en parallèle ; chaque résultat s'affiche dès qu'il est prêt
            const rowPromises = rows.map(async ([key, value]) => {
                if (value && typeof value.then === 'function') {
                    try {
                        value = await withTimeout(Promise.resolve(value), 6000, key);
                    } catch (e) {
                        value = 'Error: ' + e.message;
                    }
                }
                return [key, value];
            });

            const renderOne = async ([key, value]) => {
                full[key] = value;
                const text = await safeStringify(value);
                const display = text.length > 600
                    ? text.slice(0, 600) + '\n… (' + text.length + ' caractères — voir Copy JSON)'
                    : text;
                const row = document.createElement('div');
                row.className = 'fp-row';
                const name = document.createElement('div');
                name.className = 'fp-name';
                name.textContent = key;
                const pre = document.createElement('pre');
                pre.className = 'fp-value';
                pre.textContent = display;
                row.appendChild(name);
                row.appendChild(pre);
                container.appendChild(row);
                done++;
                status.textContent = '⏳ ' + done + '/' + total + ' attributs collectés...';
            };

            const rendered = rowPromises.map(p => p.then(renderOne).catch(() => {}));
            await Promise.all(rendered);

            const resolved = await Promise.all(rowPromises);

            // Identifiant consolidé : hash FNV-1a de toutes les valeurs collectées
            const jsonAll = JSON.stringify(resolved.map(([k, v]) => [k, v === undefined ? null : v]));
            let h = 0x811c9dc5;
            for (let i = 0; i < jsonAll.length; i++) {
                h ^= jsonAll.charCodeAt(i);
                h = Math.imul(h, 0x01000193);
            }
            const fpId = ('00000000' + (h >>> 0).toString(16)).slice(-8);
            let previousId = null;
            try {
                previousId = localStorage.getItem('__fp_demo_id');
                localStorage.setItem('__fp_demo_id', fpId);
            } catch (e) {}

            const banner = document.createElement('div');
            banner.className = 'fp-row';
            const bannerName = document.createElement('div');
            bannerName.className = 'fp-name';
            bannerName.textContent = '🆔 EMPREINTE CONSOLIDÉE (hash de tous les attributs)';
            const bannerValue = document.createElement('pre');
            bannerValue.className = 'fp-value';
            bannerValue.textContent = fpId + (previousId
                ? (previousId === fpId
                    ? '  — 🔁 visiteur reconnu (déjà vu lors d\'une visite précédente : ' + previousId + ')'
                    : '  — config modifiée depuis la dernière visite (ID précédent : ' + previousId + ')')
                : '  — première visite de votre part');
            banner.appendChild(bannerName);
            banner.appendChild(bannerValue);

            // L'empreinte consolidée reste visible en haut de la liste
            container.prepend(banner);

            try {
                window.__fpJson = JSON.stringify(full, (k, v) => v === undefined ? 'undefined' : v, 2);
            } catch (e) {
                let parts = [];
                for (const [key, value] of resolved) {
                    parts.push('"' + key + '": ' + await safeStringify(value));
                }
                window.__fpJson = '{\n' + parts.join(',\n') + '\n}';
            }
            status.textContent = '✅ ' + resolved.length + ' attributs collectés — voilà exactement ce qu\'un traqueur peut voir sur vous.';
        }

        window.runFingerprint = renderFingerprint;

        window.copyFingerprint = function() {
            const btn = document.getElementById('fp-copy-btn');
            navigator.clipboard.writeText(window.__fpJson || '').then(() => {
                const originalText = btn.textContent;
                btn.textContent = '✓ Copied!';
                setTimeout(() => { btn.textContent = originalText; }, 2000);
            }).catch(err => alert('Failed to copy: ' + err));
        };

        window.addEventListener('load', function() {
            renderFingerprint().catch(function(e) {
                const status = document.getElementById('fp-status');
                if (status) status.textContent = '❌ Erreur pendant la collecte : ' + e.message;
            });
        });
    })();
</script>

<script>
    (function() {
        function addRow(container, name, value) {
            const row = document.createElement('div');
            row.className = 'fp-row';
            const n = document.createElement('div');
            n.className = 'fp-name';
            n.textContent = name;
            const v = document.createElement('pre');
            v.className = 'fp-value';
            v.textContent = value;
            row.appendChild(n);
            row.appendChild(v);
            container.appendChild(row);
        }

        function haversineKm(lat1, lon1, lat2, lon2) {
            const R = 6371;
            const dLat = (lat2 - lat1) * Math.PI / 180;
            const dLon = (lon2 - lon1) * Math.PI / 180;
            const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
                Math.sin(dLon / 2) * Math.sin(dLon / 2);
            return 2 * R * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        }

        async function fetchIpInfo() {
            // Chaîne de fallback : ipwho.is puis ipapi.co
            try {
                const r = await fetch('https://ipwho.is/');
                const d = await r.json();
                if (d && d.success !== false && d.ip) {
                    return {
                        ip: d.ip,
                        type: d.type,
                        country: (d.country || '') + (d.flag && d.flag.emoji ? ' ' + d.flag.emoji : ''),
                        countryCode: d.country_code,
                        region: d.region,
                        city: d.city,
                        postal: d.postal,
                        lat: d.latitude,
                        lon: d.longitude,
                        timezone: d.timezone && d.timezone.id,
                        callingCode: d.calling_code,
                        isEu: d.is_eu,
                        isp: d.connection && d.connection.isp,
                        org: d.connection && d.connection.org,
                        asn: d.connection && d.connection.asn,
                        domain: d.connection && d.connection.domain
                    };
                }
            } catch (e) {}
            const r = await fetch('https://ipapi.co/json/');
            const d = await r.json();
            if (d.error) throw new Error(d.reason || 'ipapi error');
            return {
                ip: d.ip,
                type: d.version,
                country: (d.country_name || '') + ' ' + (d.country_emoji || ''),
                countryCode: d.country_code,
                region: d.region,
                city: d.city,
                postal: d.postal,
                lat: d.latitude,
                lon: d.longitude,
                timezone: d.timezone,
                callingCode: d.country_calling_code,
                isEu: d.in_eu,
                isp: d.org,
                org: d.org,
                asn: d.asn,
                domain: undefined
            };
        }

        async function renderIpInfo() {
            const status = document.getElementById('geo-status');
            const container = document.getElementById('ip-results');
            if (!status || !container) return;
            let info;
            try {
                info = await fetchIpInfo();
            } catch (e) {
                status.textContent = '❌ Impossible de résoudre l\'IP publique (' + e.message + ')';
                return;
            }
            window.__ipInfo = info;
            const maps = `https://www.openstreetmap.org/?mlat=${info.lat}&mlon=${info.lon}#map=11/${info.lat}/${info.lon}`;
            const browserTz = Intl.DateTimeFormat().resolvedOptions().timeZone;
            const tzMatch = (info.timezone && info.timezone === browserTz);
            addRow(container, 'Adresse IP publique', info.ip + ' (' + (info.type || '?') + ')');
            addRow(container, 'Pays', info.country + ' (' + (info.countryCode || '?') + ')' + (info.isEu ? ' — UE' : ''));
            addRow(container, 'Région / Ville', (info.region || '?') + ' / ' + (info.city || '?') + (info.postal ? ' (' + info.postal + ')' : ''));
            addRow(container, 'Coordonnées IP', info.lat + ', ' + info.lon + '  →  ' + maps);
            addRow(container, 'FAI / Organisation', (info.isp || '?') + (info.asn ? ' — AS' + info.asn : ''));
            addRow(container, 'Fuseau horaire IP vs navigateur', info.timezone + ' vs ' + browserTz + (tzMatch ? '  ✅ cohérent' : '  ⚠️ incohérent (VPN / proxy possible)'));
            status.textContent = '✅ Localisation IP résolue. Cliquez pour comparer avec votre position GPS réelle.';
        }

        window.requestGps = function() {
            const container = document.getElementById('gps-results');
            if (!navigator.geolocation) {
                addRow(container, 'Géolocalisation', 'API non supportée par ce navigateur');
                return;
            }
            navigator.geolocation.getCurrentPosition(function(pos) {
                const c = pos.coords;
                addRow(container, 'Position GPS précise', c.latitude + ', ' + c.longitude + ' (±' + Math.round(c.accuracy) + ' m)' + (c.altitude != null ? ' — altitude ' + Math.round(c.altitude) + ' m' : ''));
                addRow(container, 'Carte GPS', `https://www.openstreetmap.org/?mlat=${c.latitude}&mlon=${c.longitude}#map=17/${c.latitude}/${c.longitude}`);
                if (window.__ipInfo) {
                    const km = haversineKm(window.__ipInfo.lat, window.__ipInfo.lon, c.latitude, c.longitude);
                    addRow(container, 'Écart IP ↔ GPS réel', km < 15 ? km.toFixed(1) + ' km — localisation IP fiable ici' : km.toFixed(0) + ' km — vous êtes probablement derrière un VPN/proxy');
                }
            }, function(err) {
                addRow(container, 'Géolocalisation', 'Refusée ou indisponible : ' + err.message);
            }, { enableHighAccuracy: true, timeout: 10000 });
        };

        window.addEventListener('load', renderIpInfo);
    })();
</script>
