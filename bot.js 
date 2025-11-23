import makeWASocket, {
    DisconnectReason,
    useMultiFileAuthState,
    fetchLatestBaileysVersion,
    makeCacheableSignalKeyStore,
    Browsers
} from '@whiskeysockets/baileys';
import pino from 'pino';
import express from 'express';
import bodyParser from 'body-parser';
import fs from 'fs';
import path from 'path';

const app = express();
app.use(bodyParser.urlencoded({ extended: true }));

// Dossier pour sauvegarder les credentials
const AUTH_DIR = path.join(process.cwd(), 'auth_info_baileys');
if (!fs.existsSync(AUTH_DIR)) fs.mkdirSync(AUTH_DIR);

let sock;
let pairingCodeRequested = false;

// Page HTML principale
app.get('/', (req, res) => {
    res.send(`
        <html>
        <head>
            <title>WhatsApp Pairing</title>
            <style>
                body { font-family: Arial, sans-serif; text-align:center; padding: 50px; background:#f2f2f2; }
                input, button { padding: 10px; font-size:16px; margin:5px; }
                h2 { color: #333; }
                .code { font-weight:bold; font-size:20px; color:#007bff; }
                .error { color:red; font-weight:bold; }
            </style>
        </head>
        <body>
            <h2>Connexion WhatsApp - Pairing Code</h2>
            <form action="/pair" method="post">
                <input type="text" name="phone" placeholder="Numéro (code pays + numéro)" required />
                <br>
                <button type="submit">Générer le code</button>
            </form>
        </body>
        </html>
    `);
});

// Route pour générer le pairing code
app.post('/pair', async (req, res) => {
    const phoneNumber = req.body.phone.replace(/[^0-9]/g, '');
    if (phoneNumber.length < 10 || phoneNumber.length > 15) {
        return res.send(`<p class="error">❌ Numéro invalide (10-15 chiffres requis)</p>`);
    }

    if (!sock) return res.send(`<p class="error">❌ Bot non initialisé, attendre quelques secondes après démarrage</p>`);

    if (pairingCodeRequested) return res.send(`<p class="error">⚠️ Code déjà généré, vérifier WhatsApp</p>`);

    pairingCodeRequested = true;

    try {
        const code = await sock.requestPairingCode(phoneNumber);
        res.send(`
            <h3>Code de jumelage généré</h3>
            <p>📱 Entrez ce code sur WhatsApp: <span class="code">${code.toUpperCase()}</span></p>
            <p>Valide 60 secondes</p>
            <p>Étapes:</p>
            <ol>
                <li>Ouvrir WhatsApp</li>
                <li>Menu (⋮) → Appareils connectés</li>
                <li>Connecter un appareil</li>
                <li>"Connecter avec numéro de téléphone"</li>
                <li>Entrer le code affiché ci-dessus</li>
            </ol>
        `);
    } catch (err) {
        console.error(err);
        res.send(`<p class="error">❌ Erreur lors de la génération du code</p>`);
    }
});

// Fonction principale du bot
async function connectWhatsApp() {
    const { state, saveCreds } = await useMultiFileAuthState(AUTH_DIR);
    const { version, isLatest } = await fetchLatestBaileysVersion();
    console.log(`📦 Version WhatsApp Web: ${version.join('.')}`);
    console.log(`✅ Dernière version: ${isLatest ? 'Oui' : 'Non'}`);

    sock = makeWASocket({
        version,
        logger: pino({ level: 'silent' }),
        printQRInTerminal: false,
        auth: {
            creds: state.creds,
            keys: makeCacheableSignalKeyStore(state.keys, pino({ level: 'silent' }))
        },
        browser: Browsers.macOS('Chrome'),
        markOnlineOnConnect: true,
        syncFullHistory: false,
        mobile: false,
        getMessage: async (key) => ({ conversation: '' })
    });

    sock.ev.on('creds.update', saveCreds);

    sock.ev.on('connection.update', (update) => {
        const { connection, lastDisconnect } = update;
        if (connection === 'connecting') console.log('🔄 Connexion en cours...');
        if (connection === 'close') {
            const statusCode = lastDisconnect?.error?.output?.statusCode;
            const shouldReconnect = statusCode !== DisconnectReason.loggedOut;
            console.log('❌ Connexion fermée', statusCode);
            if (shouldReconnect) setTimeout(() => connectWhatsApp(), 5000);
        }
        if (connection === 'open') console.log('✅ BOT CONNECTÉ AVEC SUCCÈS!');
    });

    sock.ev.on('messages.upsert', async ({ messages, type }) => {
        if (type !== 'notify') return;
        const msg = messages[0];
        if (!msg.message) return;

        const messageText = msg.message.conversation || msg.message.extendedTextMessage?.text || '';
        const from = msg.key.remoteJid;
        const isGroup = from?.endsWith('@g.us');

        console.log(`📩 Message ${isGroup ? 'groupe' : 'privé'}: ${messageText}`);

        if (messageText.toLowerCase() === '!ping') await sock.sendMessage(from, { text: '🏓 Pong! Bot en ligne!' });
        if (messageText.toLowerCase() === '!bonjour') await sock.sendMessage(from, { text: '👋 Salut! Bot WhatsApp avec Baileys v7!' });
        if (messageText.toLowerCase() === '!help') await sock.sendMessage(from, { text: '📌 Commandes: !ping, !bonjour, !info, !help' });
        if (messageText.toLowerCase() === '!info') await sock.sendMessage(from, { text: 'ℹ️ Status: En ligne, Baileys v7.x' });
    });
}

// Démarrer le bot et le serveur web
connectWhatsApp().catch(console.error);

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`🌐 Serveur web démarré sur http://localhost:${PORT}`);
});
