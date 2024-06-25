# Django and Node.js for a video hosting mobile app 

## Detailed plan to get you started:

### 1. Project Setup

#### A. Django Setup (Backend)

1. **Install Django:**
   ```sh
   pip install django
   ```

2. **Create Django Project:**
   ```sh
   django-admin startproject video_hosting
   cd video_hosting
   ```

3. **Create Django App:**
   ```sh
   python manage.py startapp videos
   ```

4. **Configure Django Project:**
   - Add the `videos` app to `INSTALLED_APPS` in `settings.py`.
   - Configure database settings in `settings.py`.

5. **Set Up Models:**
   Define models for `User`, `Video`, and `Comment` in `videos/models.py`.
   ```python
   from django.db import models
   from django.contrib.auth.models import User

   class Video(models.Model):
       title = models.CharField(max_length=255)
       description = models.TextField()
       video_file = models.FileField(upload_to='videos/')
       uploaded_by = models.ForeignKey(User, on_delete=models.CASCADE)
       upload_date = models.DateTimeField(auto_now_add=True)

   class Comment(models.Model):
       video = models.ForeignKey(Video, related_name='comments', on_delete=models.CASCADE)
       author = models.ForeignKey(User, on_delete=models.CASCADE)
       text = models.TextField()
       created_date = models.DateTimeField(auto_now_add=True)
   ```

6. **Run Migrations:**
   ```sh
   python manage.py makemigrations
   python manage.py migrate
   ```

7. **Create Views and URLs:**
   Define views in `videos/views.py` and map URLs in `videos/urls.py`.

#### B. Node.js Setup (Backend)

1. **Install Node.js:**
   Download and install Node.js from [nodejs.org](https://nodejs.org/).

2. **Initialize Node.js Project:**
   ```sh
   mkdir video_processing
   cd video_processing
   npm init -y
   ```

3. **Install Dependencies:**
   ```sh
   npm install express ffmpeg fluent-ffmpeg aws-sdk body-parser
   ```

4. **Set Up Express Server:**
   Create `server.js` for the Node.js server.
   ```javascript
   const express = require('express');
   const bodyParser = require('body-parser');
   const ffmpeg = require('fluent-ffmpeg');
   const AWS = require('aws-sdk');
   const app = express();

   app.use(bodyParser.json());

   // AWS S3 Configuration
   const s3 = new AWS.S3({
       accessKeyId: 'YOUR_AWS_ACCESS_KEY_ID',
       secretAccessKey: 'YOUR_AWS_SECRET_ACCESS_KEY'
   });

   app.post('/upload', (req, res) => {
       const video = req.files.video;
       const uploadParams = {
           Bucket: 'YOUR_BUCKET_NAME',
           Key: video.name,
           Body: video.data
       };

       s3.upload(uploadParams, (err, data) => {
           if (err) {
               return res.status(500).json({ error: err });
           }
           res.status(200).json({ message: 'Upload successful', data });
       });
   });

   app.post('/transcode', (req, res) => {
       const inputPath = req.body.inputPath;
       const outputPath = 'transcoded_video.mp4';

       ffmpeg(inputPath)
           .output(outputPath)
           .on('end', () => {
               res.status(200).json({ message: 'Transcoding complete', outputPath });
           })
           .on('error', (err) => {
               res.status(500).json({ error: err });
           })
           .run();
   });

   const PORT = process.env.PORT || 3000;
   app.listen(PORT, () => {
       console.log(`Server running on port ${PORT}`);
   });
   ```

### 2. Integration

#### A. Communicating Between Django and Node.js

1. **Django to Node.js:**
   - Use Django's `requests` library to send HTTP requests to the Node.js server for video processing.
   - Example:
     ```python
     import requests

     def upload_video_to_node(video_path):
         url = 'http://localhost:3000/upload'
         files = {'video': open(video_path, 'rb')}
         response = requests.post(url, files=files)
         return response.json()
     ```

2. **Node.js to Django:**
   - After processing videos in Node.js, send a callback to Django to update the video status or metadata.
   - Example:
     ```javascript
     const axios = require('axios');

     function notifyDjango(videoId, status) {
         axios.post('http://localhost:8000/api/video/status', {
             videoId: videoId,
             status: status
         })
         .then(response => console.log(response.data))
         .catch(error => console.error(error));
     }
     ```

### 3. Mobile App Development

#### A. React Native Setup (Frontend)

1. **Install React Native CLI:**
   ```sh
   npm install -g react-native-cli
   ```

2. **Create React Native Project:**
   ```sh
   react-native init VideoHostingApp
   cd VideoHostingApp
   ```

3. **Install Dependencies:**
   ```sh
   npm install axios react-navigation react-native-video
   ```

4. **Set Up Navigation:**
   Create a basic navigation structure in `App.js`.
   ```javascript
   import React from 'react';
   import { NavigationContainer } from '@react-navigation/native';
   import { createStackNavigator } from '@react-navigation/stack';
   import HomeScreen from './screens/HomeScreen';
   import UploadScreen from './screens/UploadScreen';

   const Stack = createStackNavigator();

   function App() {
       return (
           <NavigationContainer>
               <Stack.Navigator initialRouteName="Home">
                   <Stack.Screen name="Home" component={HomeScreen} />
                   <Stack.Screen name="Upload" component={UploadScreen} />
               </Stack.Navigator>
           </NavigationContainer>
       );
   }

   export default App;
   ```

5. **Create Screens:**
   Define `HomeScreen` and `UploadScreen` components.

   - **HomeScreen.js:**
     ```javascript
     import React from 'react';
     import { View, Text, Button } from 'react-native';

     function HomeScreen({ navigation }) {
         return (
             <View>
                 <Text>Home Screen</Text>
                 <Button
                     title="Upload Video"
                     onPress={() => navigation.navigate('Upload')}
                 />
             </View>
         );
     }

     export default HomeScreen;
     ```

   - **UploadScreen.js:**
     ```javascript
     import React, { useState } from 'react';
     import { View, Button, Text } from 'react-native';
     import DocumentPicker from 'react-native-document-picker';
     import axios from 'axios';

     function UploadScreen() {
         const [video, setVideo] = useState(null);

         const pickVideo = async () => {
             try {
                 const res = await DocumentPicker.pick({
                     type: [DocumentPicker.types.video],
                 });
                 setVideo(res);
             } catch (err) {
                 if (DocumentPicker.isCancel(err)) {
                     console.log('User cancelled the picker');
                 } else {
                     throw err;
                 }
             }
         };

         const uploadVideo = async () => {
             if (!video) return;
             const formData = new FormData();
             formData.append('video', {
                 uri: video.uri,
                 type: video.type,
                 name: video.name,
             });

             try {
                 const response = await axios.post('http://localhost:3000/upload', formData, {
                     headers: {
                         'Content-Type': 'multipart/form-data',
                     },
                 });
                 console.log(response.data);
             } catch (error) {
                 console.error(error);
             }
         };

         return (
             <View>
                 <Button title="Pick Video" onPress={pickVideo} />
                 {video && <Text>{video.name}</Text>}
                 <Button title="Upload Video" onPress={uploadVideo} />
             </View>
         );
     }

     export default UploadScreen;
     ```

### 4. Testing and Deployment

1. **Testing:**
   - Test the Django and Node.js integration thoroughly.
   - Test video upload, processing, and playback functionalities in the mobile app.

2. **Deployment:**
   - Deploy the Django backend on a server (e.g., AWS EC2, DigitalOcean).
   - Deploy the Node.js server on a separate instance or container.
   - Use a CDN for video delivery.
   - Publish the mobile app to Google Play Store and Apple App Store.

### Conclusion

By following these steps, you can set up a robust video hosting platform using Django for the backend, Node.js for video processing, and React Native for the mobile app. Each component communicates effectively to provide a seamless user experience for video upload, processing, and playback.