# api

upload video button doesn’t have to display video on screen

it has to send that video in a post request to the end device (which is actually the device hosting the web server.. so just has to call the video file with the script)

it gets processed using the script

needs to save the NEW video to a new MP4 file? or is it the same mp4 file and then it overwrites

either one it needs to save that new mp4, the data, and the file location of the new mp4/

[https://dev.to/sjamescarter/uploading-images-in-react-3lmb](https://dev.to/sjamescarter/uploading-images-in-react-3lmb)

‘react js upload image to server’

```jsx
npm install axios
```

upload.js

```jsx
import React, { useState } from "react";
import axios from "axios";

export default function Upload() {
  const [file, setFile] = useState(null);
  const [status, setStatus] = useState("");

  const handleFileChange = (e) => {
    setFile(e.target.files[0]);
  };

  const handleUpload = async () => {
    if (!file) {
      setStatus("Please select a file first");
      return;
    }

    const formData = new FormData();
    formData.append("video", file);

    try {
      setStatus("Uploading...");
      const res = await axios.post("http://localhost:5000/upload", formData, {
        headers: { "Content-Type": "multipart/form-data" },
      });
      setStatus(`Success: ${res.data.message}`);
    } catch (err) {
      setStatus("Upload failed");
    }
  };

  return (
    <div className="p-4">
      <input type="file" accept="video/*" onChange={handleFileChange} />
      <button onClick={handleUpload} className="ml-2 bg-blue-500 text-white px-4 py-2 rounded">
        Upload
      </button>
      <p>{status}</p>
    </div>
  );
}

```

```jsx
node server.js
npm start
```

```jsx
const tempPath = req.file.path;
const targetPath = path.join(__dirname, "processed", req.file.originalname);

fs.rename(tempPath, targetPath, (err) => {
  if (err) return res.status(500).json({ error: "Processing failed" });

  console.log("File processed:", targetPath);
  res.json({ message: "Upload successful", file: req.file.originalname });
});

```

How Multer works:

### 1. **Initial Upload (temporary folder)**

```jsx
const upload = multer({
  dest: "uploads/",
});
```

- Multer writes the uploaded file into the `uploads/` directory inside your project.
- The file at this point will have a **random filename** (not the original filename).
- Example:
    
    ```
    project-root/
    ├── server.js
    ├── uploads/
    │   └── 4c8a2f99d0f1f84c4e0f9e9b   ← temp file
    ```
    

### 2. **Processing / Moving**

In the `app.post("/upload")` route, I included this code:

```jsx
const tempPath = req.file.path;
const targetPath = path.join(__dirname, "processed", req.file.originalname);

fs.rename(tempPath, targetPath, (err) => {
  if (err) return res.status(500).json({ error: "Processing failed" });

  console.log("File processed:", targetPath);
  res.json({ message: "Upload successful", file: req.file.originalname });
});
```

- This moves the file from `uploads/` into a new `processed/` folder.
- It also renames the file back to its **original filename**.
- After upload + move, you’ll see:
    
    ```
    project-root/
    ├── server.js
    ├── uploads/          ← might still hold temp files if move fails
    ├── processed/
    │   └── myvideo.mp4   ← final saved file
    ```
    

---

👉 So the final location is:

- `processed/` folder (with original name), after successful handling.
- If something goes wrong before `fs.rename`, the file will remain in the `uploads/` folder under a random name.

////////////////////////////////////////////////////////////////////////////////////////////////////////////////

updated code (creates processed folder if it doesn’t exist):

```jsx
const express = require("express");
const multer = require("multer");
const cors = require("cors");
const path = require("path");
const fs = require("fs");

const app = express();
app.use(cors()); // Allow frontend requests

// Configure multer for video uploads
const upload = multer({
  dest: "uploads/", // temporary storage
  limits: { fileSize: 100 * 1024 * 1024 }, // 100 MB limit
});

// Ensure the "processed" folder exists
const processedDir = path.join(__dirname, "processed");
if (!fs.existsSync(processedDir)) {
  fs.mkdirSync(processedDir, { recursive: true });
}

// Upload endpoint
app.post("/upload", upload.single("video"), (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: "No file uploaded" });
  }

  console.log("File uploaded:", req.file);

  const tempPath = req.file.path;
  const targetPath = path.join(processedDir, req.file.originalname);

  fs.rename(tempPath, targetPath, (err) => {
    if (err) {
      console.error("Error processing file:", err);
      return res.status(500).json({ error: "Processing failed" });
    }

    console.log("File saved to:", targetPath);
    res.json({ message: "Upload successful", file: req.file.originalname });
  });
});

// Start server
const PORT = 5000;
app.listen(PORT, () => console.log(`Server running on http://localhost:${PORT}`));

```

this was added:

```jsx
const processedDir = path.join(__dirname, "processed");
if (!fs.existsSync(processedDir)) {
  fs.mkdirSync(processedDir, { recursive: true });
}
```

Changed the `targetPath` to use this guaranteed `processedDir`.

////////////////////////////////////////////////////////////////////////////////////////////////////////////////

server.js (add a new endpoint)

```jsx
// List all processed videos
app.get("/videos", (req, res) => {
  fs.readdir(processedDir, (err, files) => {
    if (err) {
      console.error("Error reading processed folder:", err);
      return res.status(500).json({ error: "Could not read processed folder" });
    }

    // Return the list of files with URLs
    const fileList = files.map((file) => ({
      name: file,
      url: `http://localhost:${PORT}/processed/${file}`,
    }));

    res.json(fileList);
  });
});

// Serve processed videos as static files
app.use("/processed", express.static(processedDir));
```

### What this does:

1. **`/videos`** → returns a JSON array of video files with their URLs:
    
    ```json
    [
      { "name": "example.mp4", "url": "http://localhost:5000/processed/example.mp4" }
    ]
    ```
    
2. **`/processed/...`** → serves videos directly so you can stream or download them.

frontend component to list all videos after upload:

```jsx
import React, { useState, useEffect } from "react";
import axios from "axios";

export default function VideoGallery() {
  const [videos, setVideos] = useState([]);

  useEffect(() => {
    const fetchVideos = async () => {
      try {
        const res = await axios.get("http://localhost:5000/videos");
        setVideos(res.data);
      } catch (err) {
        console.error("Error fetching videos", err);
      }
    };
    fetchVideos();
  }, []);

  return (
    <div className="p-4">
      <h2 className="text-xl font-bold mb-2">Uploaded Videos</h2>
      {videos.length === 0 ? (
        <p>No videos uploaded yet.</p>
      ) : (
        <ul className="space-y-4">
          {videos.map((video) => (
            <li key={video.name} className="border p-2 rounded shadow">
              <p className="font-medium">{video.name}</p>
              <video controls width="400">
                <source src={video.url} type="video/mp4" />
                Your browser does not support the video tag.
              </video>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

- `/upload` → upload a video.
- `/videos` → list all processed videos.
- `/processed/...` → stream the actual video files.
- React gallery that displays the uploaded videos with a video player.

////////////////////////////////////////////////////////////////////////////////////////////////////////////////

updated frontend so don’t have to refresh page to get videos:

```jsx
import React, { useState, useEffect } from "react";
import axios from "axios";

export default function VideoUploader() {
  const [file, setFile] = useState(null);
  const [status, setStatus] = useState("");
  const [videos, setVideos] = useState([]);

  // Fetch video list from backend
  const fetchVideos = async () => {
    try {
      const res = await axios.get("http://localhost:5000/videos");
      setVideos(res.data);
    } catch (err) {
      console.error("Error fetching videos", err);
    }
  };

  useEffect(() => {
    fetchVideos();
  }, []);

  const handleFileChange = (e) => {
    setFile(e.target.files[0]);
  };

  const handleUpload = async () => {
    if (!file) {
      setStatus("Please select a file first");
      return;
    }

    const formData = new FormData();
    formData.append("video", file);

    try {
      setStatus("Uploading...");
      await axios.post("http://localhost:5000/upload", formData, {
        headers: { "Content-Type": "multipart/form-data" },
      });
      setStatus("Upload successful ✅");
      setFile(null);
      fetchVideos(); // refresh video list after upload
    } catch (err) {
      console.error(err);
      setStatus("Upload failed ❌");
    }
  };

  return (
    <div className="p-6 space-y-6">
      {/* Upload Form */}
      <div className="space-x-2">
        <input type="file" accept="video/*" onChange={handleFileChange} />
        <button
          onClick={handleUpload}
          className="bg-blue-500 text-white px-4 py-2 rounded"
        >
          Upload
        </button>
        <p>{status}</p>
      </div>

      {/* Video Gallery */}
      <div>
        <h2 className="text-xl font-bold mb-4">Uploaded Videos</h2>
        {videos.length === 0 ? (
          <p>No videos uploaded yet.</p>
        ) : (
          <ul className="space-y-6">
            {videos.map((video) => (
              <li key={video.name} className="border p-3 rounded shadow">
                <p className="font-medium mb-2">{video.name}</p>
                <video controls width="400">
                  <source src={video.url} type="video/mp4" />
                  Your browser does not support the video tag.
                </video>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
}
```

////////////////////////////////////////////////////////////////////////////////////////////////////////////////

slightly optimized backend (one less API call):

```jsx
app.post("/upload", upload.single("video"), (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: "No file uploaded" });
  }

  console.log("File uploaded:", req.file);

  const tempPath = req.file.path;
  const targetPath = path.join(processedDir, req.file.originalname);

  fs.rename(tempPath, targetPath, (err) => {
    if (err) {
      console.error("Error processing file:", err);
      return res.status(500).json({ error: "Processing failed" });
    }

    console.log("File saved to:", targetPath);

    // Return updated list of videos after upload
    fs.readdir(processedDir, (err, files) => {
      if (err) {
        return res.status(500).json({ error: "Could not read processed folder" });
      }

      const fileList = files.map((file) => ({
        name: file,
        url: `http://localhost:${PORT}/processed/${file}`,
      }));

      res.json({
        message: "Upload successful",
        videos: fileList,
      });
    });
  });
});
```

Now the frontend can update the gallery directly from the `/upload` response:

```jsx
const handleUpload = async () => {
  if (!file) {
    setStatus("Please select a file first");
    return;
  }

  const formData = new FormData();
  formData.append("video", file);

  try {
    setStatus("Uploading...");
    const res = await axios.post("http://localhost:5000/upload", formData, {
      headers: { "Content-Type": "multipart/form-data" },
    });

    setStatus("Upload successful ✅");
    setFile(null);
    setVideos(res.data.videos); // update directly from response
  } catch (err) {
    console.error(err);
    setStatus("Upload failed ❌");
  }
};

```

### What changed:

- Backend `/upload` now responds with the **full updated video list**.
- Frontend uses `res.data.videos` to refresh the gallery immediately.
- No need to call `/videos` again after uploading

////////////////////////////////////////////////////////////////////////////////////////////////////////////////

delete function:
Add this new route **below** your `/upload` and `/videos` routes:

server.js

```jsx
// Delete a video by filename
app.delete("/videos/:filename", (req, res) => {
  const filename = req.params.filename;
  const filePath = path.join(processedDir, filename);

  fs.unlink(filePath, (err) => {
    if (err) {
      console.error("Error deleting file:", err);
      return res.status(500).json({ error: "Could not delete file" });
    }

    console.log("Deleted:", filename);

    // Return updated video list after deletion
    fs.readdir(processedDir, (err, files) => {
      if (err) {
        return res.status(500).json({ error: "Could not read processed folder" });
      }

      const fileList = files.map((file) => ({
        name: file,
        url: `http://localhost:${PORT}/processed/${file}`,
      }));

      res.json({
        message: "File deleted",
        videos: fileList,
      });
    });
  });
});
```

add delete button for each video:

```jsx
const handleDelete = async (filename) => {
  try {
    const res = await axios.delete(`http://localhost:5000/videos/${filename}`);
    setVideos(res.data.videos); // update gallery after deletion
  } catch (err) {
    console.error("Delete failed", err);
  }
};
```

update gallery list in the component:

```jsx
<ul className="space-y-6">
  {videos.map((video) => (
    <li key={video.name} className="border p-3 rounded shadow">
      <div className="flex justify-between items-center mb-2">
        <p className="font-medium">{video.name}</p>
        <button
          onClick={() => handleDelete(video.name)}
          className="bg-red-500 text-white px-3 py-1 rounded"
        >
          Delete
        </button>
      </div>
      <video controls width="400">
        <source src={video.url} type="video/mp4" />
        Your browser does not support the video tag.
      </video>
    </li>
  ))}
</ul>
```

this now does:

- `/upload` → upload video (returns updated list).
- `/videos` → fetch list of videos.
- `/processed/...` → serve videos.
- `/videos/:filename (DELETE)` → delete a video and return updated list.
- React: Upload + Gallery + Delete.