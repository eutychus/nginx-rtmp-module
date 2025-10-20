# Exampled

### Simple Video-on-Demand

```sh
rtmp {
    server {
        listen 1935;
        application vod {
            play /var/flvs;
        }
    }
}
```

### Simple live broadcast service
```sh
rtmp {
    server {
        listen 1935;
        application live {
            live on;
        }
    }
}
```

### Re-translate remote stream
```sh
rtmp {
    server {
        listen 1935;
        application tv {
            live on;
            pull rtmp://cdn.example.com:443/programs/main pageUrl=http://www.example.com/index.html name=maintv;
        }
    }
}
```

### Re-translate remote stream with HLS support
```sh
rtmp {
    server {
        listen 1935;
        application tv {
            live on;
            hls on;
            hls_path /tmp/tv2;
            hls_fragment 15s;

            pull rtmp://tv2.example.com:443/root/new name=tv2;
        }
    }
}
http {
    server {
        listen 80;
        location /tv2 {
            alias /tmp/tv2;
        }
    }
}
```

### Stream your X screen through RTMP
```sh
$ ffmpeg -f x11grab -follow_mouse centered -r 25 -s cif -i :0.0 -f flv rtmp://localhost/myapp/screen
```

### HLS with segment notifications for cloud storage upload
This example demonstrates how to use the `on_hls_segment` callback to automatically
upload HLS segments to cloud storage (e.g., AWS S3, Google Cloud Storage) as they are created.

```sh
rtmp {
    server {
        listen 1935;
        application live {
            live on;
            
            # Enable HLS
            hls on;
            hls_path /tmp/hls;
            hls_fragment 5s;
            hls_playlist_length 30s;
            hls_nested on;
            hls_cleanup on;
            
            # Notify backend service when new segments are created
            on_hls_segment http://localhost:8080/segment-upload;
            
            # Optional: notify on publish events
            on_publish http://localhost:8080/on-publish;
            on_publish_done http://localhost:8080/on-publish-done;
        }
    }
}

http {
    server {
        listen 8080;
        
        # Serve HLS content to players
        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /tmp;
            add_header Cache-Control no-cache;
            add_header Access-Control-Allow-Origin *;
        }
        
        # Handle segment upload notifications
        location /segment-upload {
            # Forward to your backend service for processing
            proxy_pass http://backend-service:3000/upload-segment;
            proxy_set_header Content-Type application/x-www-form-urlencoded;
        }
    }
}
```

Example backend service (Node.js with Express) to handle segment uploads:

```javascript
const express = require('express');
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
const fs = require('fs').promises;
const path = require('path');

const app = express();
app.use(express.urlencoded({ extended: true }));

const s3 = new S3Client({ region: 'us-east-1' });

app.post('/upload-segment', async (req, res) => {
    try {
        const {
            module,      // "hls" or "dash"
            name,        // stream name
            segment,     // segment file path
            playlist,    // playlist file path
            sequence,    // segment sequence number
            duration,    // segment duration in seconds
            video_codec, // e.g., "H264"
            audio_codec, // e.g., "AAC"
        } = req.body;
        
        console.log(`New segment created: ${segment}`);
        console.log(`Sequence: ${sequence}, Duration: ${duration}s`);
        
        // Read segment file
        const segmentData = await fs.readFile(segment);
        const fileName = path.basename(segment);
        
        // Upload to S3
        await s3.send(new PutObjectCommand({
            Bucket: 'my-hls-bucket',
            Key: `streams/${name}/${fileName}`,
            Body: segmentData,
            ContentType: 'video/mp2ts',
            CacheControl: 'max-age=31536000', // 1 year for segments
        }));
        
        // Also upload playlist
        if (playlist) {
            const playlistData = await fs.readFile(playlist);
            const playlistName = path.basename(playlist);
            
            await s3.send(new PutObjectCommand({
                Bucket: 'my-hls-bucket',
                Key: `streams/${name}/${playlistName}`,
                Body: playlistData,
                ContentType: 'application/vnd.apple.mpegurl',
                CacheControl: 'no-cache', // Don't cache playlists
            }));
        }
        
        console.log(`Successfully uploaded segment to S3: ${fileName}`);
        res.status(200).send('OK');
        
    } catch (error) {
        console.error('Error uploading segment:', error);
        res.status(500).send('Error');
    }
});

app.listen(3000, () => {
    console.log('Segment upload service listening on port 3000');
});
```

Example Python backend service (Flask) for segment processing:

```python
from flask import Flask, request
import boto3
import os

app = Flask(__name__)
s3_client = boto3.client('s3')

@app.route('/upload-segment', methods=['POST'])
def upload_segment():
    try:
        module = request.form.get('module')
        name = request.form.get('name')
        segment = request.form.get('segment')
        playlist = request.form.get('playlist')
        sequence = request.form.get('sequence')
        duration = request.form.get('duration')
        
        print(f"New {module} segment: {segment}")
        print(f"Sequence: {sequence}, Duration: {duration}s")
        
        # Upload segment to S3
        with open(segment, 'rb') as f:
            s3_client.put_object(
                Bucket='my-hls-bucket',
                Key=f'streams/{name}/{os.path.basename(segment)}',
                Body=f,
                ContentType='video/mp2ts',
                CacheControl='max-age=31536000'
            )
        
        # Upload playlist
        if playlist and os.path.exists(playlist):
            with open(playlist, 'rb') as f:
                s3_client.put_object(
                    Bucket='my-hls-bucket',
                    Key=f'streams/{name}/{os.path.basename(playlist)}',
                    Body=f,
                    ContentType='application/vnd.apple.mpegurl',
                    CacheControl='no-cache'
                )
        
        print(f"Successfully uploaded to S3")
        return 'OK', 200
        
    except Exception as e:
        print(f"Error: {e}")
        return 'Error', 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```

To test this setup:

1. Start your backend service (Node.js or Python)
2. Start nginx with the configuration above
3. Publish a stream:
   ```sh
   ffmpeg -re -i input.mp4 -c:v libx264 -c:a aac -f flv rtmp://localhost/live/mystream
   ```
4. Watch the logs to see segments being uploaded to cloud storage
5. Segments will be automatically uploaded as they are created

The segment notification includes rich metadata that can be used for:
- Analytics and monitoring (bitrates, codecs, resolution)
- Quality of Service tracking (segment duration, frame rate)
- Conditional processing (different handling for different codecs or bitrates)
- Multi-CDN distribution strategies

