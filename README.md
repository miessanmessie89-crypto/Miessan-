<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ONYX SCAN - ABIDJAN CORE</title>
    <script src="https://unpkg.com/@zxing/library@latest"></script>
    <style>
        body {
            background-color: #000;
            color: #fff;
            font-family: 'Segoe UI', sans-serif;
            margin: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            height: 100vh;
            overflow: hidden;
        }

        /* HEADER STYLE ABIDJAN CORE */
        header {
            width: 100%;
            padding: 15px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            background: #111;
            border-bottom: 2px solid #00FF00;
        }

        .logo-text { font-weight: bold; color: #00FF00; font-size: 20px; }
        .badge { background: #00FF00; color: #000; padding: 2px 10px; border-radius: 20px; font-size: 12px; font-weight: bold; }

        /* ZONE VIDÉO DU SCANNER */
        #scanner-container {
            width: 90%;
            height: 40%;
            margin-top: 20px;
            border: 3px solid #00FF00;
            position: relative;
            border-radius: 15px;
            overflow: hidden;
        }

        video { width: 100%; height: 100%; object-fit: cover; }

        /* LIGNE ROUGE DE SCAN */
        .scan-line {
            position: absolute;
            top: 50%; width: 100%; height: 2px;
            background: red; box-shadow: 0 0 10px red;
            animation: move 2s infinite ease-in-out;
        }

        @keyframes move {
            0%, 100% { top: 20%; }
            50% { top: 80%; }
        }

        /* AFFICHAGE DU RÉSULTAT */
        #lisa-output {
            width: 90%;
            background: #111;
            margin-top: 20px;
            padding: 20px;
            border-radius: 15px;
            border-left: 5px solid #00FF00;
            min-height: 100px;
            font-size: 18px;
        }

        .status-dot { height: 10px; width: 10px; background-color: #00FF00; border-radius: 50%; display: inline-block; margin-right: 10px; }
    </style>
</head>
<body>

    <header>
        <div class="logo-text">ONYX <span style="color:#fff">SCAN</span></div>
        <div class="badge">ABIDJAN CORE</div>
    </header>

    <div id="scanner-container">
        <video id="video"></video>
        <div class="scan-line"></div>
    </div>

    <div id="lisa-output">
        <span class="status-dot"></span> PRÊT À SCANNER...
    </div>

    <audio id="beep" src="https://www.soundjay.com/buttons/beep-07.mp3"></audio>

    <script>
        const codeReader = new ZXing.BrowserBarcodeReader();
        let isLocked = false;

        async function startScanner() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } });
                document.getElementById('video').srcObject = stream;
                
                // Allumage automatique du flash si possible
                const track = stream.getVideoTracks()[0];
                setTimeout(async () => {
                    const caps = track.getCapabilities();
                    if (caps.torch) await track.applyConstraints({advanced: [{torch: true}]});
                }, 1000);

                codeReader.decodeFromVideoDevice(null, 'video', (result) => {
                    if (result && !isLocked) {
                        isLocked = true;
                        playFeedback();
                        identifierProduit(result.text);
                    }
                });
            } catch (err) { document.getElementById('lisa-output').innerText = "Erreur caméra !"; }
        }

        function playFeedback() {
            document.getElementById('beep').play();
            if(navigator.vibrate) navigator.vibrate(100);
        }

        async function identifierProduit(code) {
            const output = document.getElementById('lisa-output');
            output.innerHTML = "<em>Recherche du produit... " + code + "</em>";

            try {
                // On utilise la base de données mondiale pour le nom
                const response = await fetch(`https://world.openfoodfacts.org/api/v0/product/${code}.json`);
                const data = await response.json();

                if (data.status === 1) {
                    const nom = data.product.product_name || "PRODUIT INCONNU";
                    const marque = data.product.brands || "MARQUE INCONNUE";
                    const quantite = data.product.quantity || "";
                    
                    output.innerHTML = `
                        <div style="color:#00FF00; font-weight:bold;">${marque.toUpperCase()}</div>
                        <div style="font-size:22px;">${nom}</div>
                        <div style="color:#aaa;">${quantite}</div>
                    `;
                } else {
                    output.innerHTML = "CODE DÉTECTÉ : " + code + "<br><span style='color:orange;'>PRODUIT NON RÉPERTORIÉ</span>";
                }
            } catch (e) { output.innerText = "ERREUR RÉSEAU"; }

            // Déverrouille après 4 secondes pour le prochain scan
            setTimeout(() => { isLocked = false; }, 4000);
        }

        window.onload = startScanner;
    </script>
</body>
</html>
