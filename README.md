// server.js
// Words Of Wisdom — Live Fellowship token server
//
// This server's ONLY job is to safely generate short-lived LiveKit
// access tokens for people joining a call. Your real API secret lives
// ONLY here (as environment variables on Render), never in the
// Blogger page, never in the browser, never in chat.

const express = require('express');
const cors = require('cors');
const { AccessToken } = require('livekit-server-sdk');

const app = express();
app.use(cors()); // allows your Blogger page (a different domain) to call this server
app.use(express.json());

// ---- Config (set these as Environment Variables on Render — never hardcode) ----
const LIVEKIT_URL = process.env.LIVEKIT_URL;         // e.g. wss://glorious-fellowship-2n3zec2f.livekit.cloud
const LIVEKIT_API_KEY = process.env.LIVEKIT_API_KEY; // your rotated API key
const LIVEKIT_API_SECRET = process.env.LIVEKIT_API_SECRET; // your rotated API secret

if (!LIVEKIT_URL || !LIVEKIT_API_KEY || !LIVEKIT_API_SECRET) {
  console.warn('WARNING: LiveKit environment variables are not fully set. Set LIVEKIT_URL, LIVEKIT_API_KEY, LIVEKIT_API_SECRET in Render.');
}

const ROOM_NAME = 'live-fellowship'; // the single recurring fellowship room

// Health check — useful to confirm the server is alive
app.get('/', (req, res) => {
  res.send('Words Of Wisdom Live Fellowship token server is running.');
});

// POST /token  { name: "Favor Mewel" }
// Returns a short-lived token + the LiveKit URL the frontend needs to connect.
app.post('/token', async (req, res) => {
  try {
    const { name } = req.body;

    if (!name || typeof name !== 'string' || name.trim().length === 0) {
      return res.status(400).json({ error: 'A display name is required to join.' });
    }

    // Basic cleanup — keep names short and safe
    const safeName = name.trim().slice(0, 40);
    const identity = `${safeName}-${Math.random().toString(36).slice(2, 8)}`;

    const at = new AccessToken(LIVEKIT_API_KEY, LIVEKIT_API_SECRET, {
      identity,
      name: safeName,
      ttl: '2h', // token expires after 2 hours — plenty for one fellowship session
    });

    at.addGrant({
      room: ROOM_NAME,
      roomJoin: true,
      canPublish: true,
      canSubscribe: true,
    });

    const token = await at.toJwt();

    res.json({
      token,
      url: LIVEKIT_URL,
      room: ROOM_NAME,
    });
  } catch (err) {
    console.error('Token generation failed:', err);
    res.status(500).json({ error: 'Could not generate access token.' });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Fellowship token server listening on port ${PORT}`);
});
{
  "name": "fellowship-token-server",
  "version": "1.0.0",
  "description": "Signaling/token server for Words Of Wisdom Live Fellowship (LiveKit)",
  "main": "server.js",
  "type": "commonjs",
  "scripts": {
    "start": "node server.js"
  },
  "engines": {
    "node": ">=18.0.0"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "express": "^4.19.2",
    "livekit-server-sdk": "^2.9.2"
  }
}
# Words Of Wisdom — Live Fellowship Token Server

This is the small server that lets your Blogger "Live Fellowship" page
connect people into a real-time voice/video call, powered by LiveKit.

It does ONE job: safely hand out short-lived access tokens. Your real
LiveKit API secret lives only here, as environment variables — never
in the Blogger page, never in a browser, never in chat.

## 1. Rotate your LiveKit keys (if you haven't already)

If you ever pasted your API secret anywhere insecure (chat, email, etc.),
go to your LiveKit Cloud project → Settings → API Keys → regenerate.
Use the NEW key/secret below.

## 2. Deploy to Render (free tier)

1. Go to https://render.com and sign in (GitHub login is easiest).
2. Push this folder to a new GitHub repo (or use Render's "Deploy without Git"
   option if offered — otherwise create a repo named e.g. `fellowship-token-server`
   and push these files to it).
3. In Render: New → Web Service → connect your repo.
4. Settings:
   - **Runtime:** Node
   - **Build Command:** `npm install`
   - **Start Command:** `npm start`
   - **Instance Type:** Free
5. Under "Environment," add these three variables:
   - `LIVEKIT_URL` = `wss://glorious-fellowship-2n3zec2f.livekit.cloud`
   - `LIVEKIT_API_KEY` = (your rotated API key)
   - `LIVEKIT_API_SECRET` = (your rotated API secret)
6. Click "Create Web Service." Render will build and deploy it.
7. Once live, Render gives you a URL like:
   `https://fellowship-token-server.onrender.com`

Test it works by visiting that URL in a browser — you should see:
"Words Of Wisdom Live Fellowship token server is running."

## 3. What's next

Once this is deployed and you give me the Render URL, I'll build the
frontend call page (join screen, video grid, mute/camera buttons) that
calls `POST https://your-render-url.onrender.com/token` to get a token,
then connects to LiveKit and joins the `live-fellowship` room — all
embedded directly in your Blogger `live-fellowship.html` page, no
redirect, no new tab.

## Notes

- Free Render web services "spin down" after periods of inactivity and
  take ~30-50 seconds to wake up on the next request. For a scheduled
  fellowship time, this is a minor delay the first person to join might
  notice — nothing breaks, it just takes a moment to wake up.
- The room name is fixed as `live-fellowship` — everyone who requests a
  token joins the same recurring room, matching your one ongoing
  fellowship meeting.
  
