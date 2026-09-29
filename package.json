const express = require('express');
const fs = require('fs');
const path = require('path');
const os = require('os');
const crypto = require('crypto');
const { spawn } = require('child_process');

const app = express();
app.use(express.json());

app.get('/', (req, res) => {
  res.send('Video render server is running');
});

app.post('/render', async (req, res) => {
  const { video_url, audio_url } = req.body;

  if (!video_url) {
    return res.status(400).json({ error: 'video_url is required' });
  }

  const id = crypto.randomBytes(6).toString('hex');
  const inputPath = path.join(os.tmpdir(), `in-${id}.mp4`);
  const audioPath = path.join(os.tmpdir(), `audio-${id}.mp3`);
  const outputPath = path.join(os.tmpdir(), `out-${id}.mp4`);

  try {
    const videoResp = await fetch(video_url);
    if (!videoResp.ok) throw new Error(`Failed to download video: ${videoResp.status}`);
    fs.writeFileSync(inputPath, Buffer.from(await videoResp.arrayBuffer()));

    let hasAudio = false;
    if (audio_url) {
      const audioResp = await fetch(audio_url);
      if (!audioResp.ok) throw new Error(`Failed to download audio: ${audioResp.status}`);
      fs.writeFileSync(audioPath, Buffer.from(await audioResp.arrayBuffer()));
      hasAudio = true;
    }

    const filter = 'scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920';

    const args = hasAudio
      ? ['-y', '-i', inputPath, '-i', audioPath,
         '-vf', filter,
         '-map', '0:v:0', '-map', '1:a:0',
         '-preset', 'veryfast', '-crf', '23',
         '-t', '30',
         '-c:v', 'libx264', '-c:a', 'aac',
         '-shortest',
         outputPath]
      : ['-y', '-i', inputPath,
         '-vf', filter,
         '-preset', 'veryfast', '-crf', '23',
         '-t', '30',
         '-c:v', 'libx264', '-c:a', 'aac',
         outputPath];

    await new Promise((resolve, reject) => {
      const ff = spawn('ffmpeg', args);
      let stderr = '';
      ff.stderr.on('data', (d) => { stderr += d.toString(); });
      ff.on('close', (code) => {
        if (code === 0) resolve();
        else reject(new Error(`ffmpeg exited with code ${code}: ${stderr.slice(-1000)}`));
      });
    });

    res.setHeader('Content-Type', 'video/mp4');
    fs.createReadStream(outputPath).pipe(res).on('close', () => {
      [inputPath, audioPath, outputPath].forEach(p => fs.existsSync(p) && fs.unlinkSync(p));
    });

  } catch (err) {
    [inputPath, audioPath, outputPath].forEach(p => fs.existsSync(p) && fs.unlinkSync(p));
    res.status(500).json({ error: err.message });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server listening on port ${PORT}`));
