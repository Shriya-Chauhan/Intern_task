# Building a Live Proctoring System using Dyte

## TL;DR: 
🔒 In this tutorial, we'll create a powerful **"Live Proctoring System"** using **Dyte APIs**. It empowers admins to monitor students in real-time and automatically detects any cheating attempts during online exams. Get ready to outsmart the cheaters! 🕵️‍♀️🔍💻

## Introduction:
🎓 **Proctoring** is an essential method to prevent cheating during exams. While offline exams have dedicated proctors, online exams require a different approach. Enter live automatic proctoring—a system that monitors students using webcams and microphones, powered by computer vision and machine learning. In this tutorial, we'll build a live proctoring system using **Dyte APIs**. With real-time monitoring, admins can swiftly detect cheating attempts and issue warning messages to ensure exam integrity. Let's dive in and outsmart the cheaters! 🚀💪💡


## High-Level Design of the Application⚡

**Frontend:**
- Utilizing React with Dyte UI kit and Dyte React Core packages for the user interface.

**Backend:**
- Employing FastAPI with Python for the backend implementation.

**Database:**
- Utilizing ElephantSQL as a Database as a Service solution for efficient data management.

**Image Storage:**
- Utilizing Imgur for storing screenshots captured during the monitoring process.

**Metadata:**
- Saving image metadata securely in the same database.

 This design ensures a user-friendly interface, efficient data management, and reliable storage of screenshots for the live proctoring system.
<img src='https://github.com/ankiiitraj/dyting/blob/main/hld-proc.png?raw=true' >

## Step 0: Configurations and Setup 🔧

Before starting the proctoring system development, follow these steps to set up the necessary configurations and tools:

1. **Dyte Account Setup**
   - Create a free account on [Dyte.io](https://dyte.io) by clicking "Start Building" and signing up with Google or GitHub.
   - Access your Dyte API keys from the "API Keys" tab in the left sidebar and keep them secure for later use.

2. **Frontend and Backend Frameworks**
   - Use React for the frontend, a JavaScript library for building user interfaces.
   - Utilize FastAPI as the backend framework, a Python framework for building APIs.

3. **Creating the Project Directory**
   - Open your terminal or command prompt and execute the following commands:
     ```
     mkdir dyte-proctoring
     cd dyte-proctoring
     ```

4. **Imagur Account Setup**
   - Sign up for an account on [Imagur](https://imagur.com) and obtain the required API key. Refer to a step-by-step guide for detailed instructions.

5. **ElephantSQL Account Setup**
   - Create an account on [ElephantSQL](https://www.elephantsql.com) and follow the step-by-step guide to set up the necessary database.

Now you are ready to proceed with building the proctoring system. 
## Step 1: Setting up the frontend ✨💻
To get started with the frontend of our live proctoring system, follow these steps:
1. Create a new React app using `create-react-app` by running the following command:
  ```bash
   yarn create react-app frontend --template typescript
   ```
   This command will set up a new React app in the `frontend` directory.

2. Install the necessary dependencies by running the following command:
   ```bash
   yarn add @dytesdk/react-web-core @dytesdk/react-ui-kit react-router react-router-dom
   ```
   These dependencies include the Dyte React Web Core, React UI Kit, and React Router packages.

3. Replace the contents of `frontend/src/App.jsx` with the following code:

   ```jsx
   import { useEffect, useState } from 'react';
   import Meet from './Meet';
   import Home from './Home';
   import { BrowserRouter, Routes, Route } from 'react-router-dom';
   import './App.css';

   function App() {
       const [meetingId, setMeetingId] = useState();

       const createMeeting = async () => {
           const resp = await fetch('http://localhost:8000/meetings', {
               method: 'POST',
               body: JSON.stringify({ title: 'Joint Entrance Examination' }),
               headers: { 'Content-Type': 'application/json' }
           });
           const respJson = await resp.json();
           console.log(respJson);
           window.localStorage.setItem('adminId', respJson.admin_id);
           setMeetingId(respJson.data.id);
       };

       useEffect(() => {
           const id = window.location.pathname.split('/')[2];
           if (!id) {
               createMeeting();
           }
       }, []);

       return (
           <BrowserRouter>
               <Routes>
                   <Route path='/' element={<Home meetingId={meetingId} />} />
                   <Route path='/meeting/:meetingId' element={<Meet />} />
               </Routes>
           </BrowserRouter>
       );
   }

   export default App;
   ```

   This code sets up the main `App` component for our application. It imports the necessary dependencies, including `Meet` and `Home` components. It also uses `react-router` to define the routes for our app. The `createMeeting` function is responsible for creating a Dyte meeting and storing the `adminId` in local storage. The `useEffect` hook ensures that a meeting is created if the `id` is not present in the URL.

   This component provides the foundation for our live proctoring system's frontend.
   
**Home Component**

The 🏠Home component is responsible for rendering the `/` route. Create a new file `frontend/src/Home.jsx` and add the following code:

```jsx
import { Link } from "react-router-dom";

function Home({ meetingId }) {
    return (
        <div style={{
            height: "100vh",
            width: "100vw",
            fontSize: "x-large",
            display: "flex",
            justifyContent: "center",
            alignItems: "center"}}
        >
            {(meetingId && !window.location.pathname.split('/')[2]) && 
                <Link to={`/meeting/${meetingId}`}>Create and Join Meeting</Link>
            }
        </div>
    )
}

export default Home;
```

The Home component renders the `/` route and displays a user-friendly interface. It consists of a centered message and a link to create and join a meeting. The link is only displayed if a `meetingId` is provided and the current URL doesn't have a meeting ID.


**🤝 Meet Component**

The Meet component plays a crucial role in our application, handling the `/meeting/:meetingId` route. It empowers us to conduct seamless meetings while distinguishing between admin and regular user roles. Let's dive into how it works:

👩‍💼 When an admin clicks on the link provided on the `/` route, they are instantly redirected to the meeting page.

🔐 The admin is automatically added to the meeting as a participant with the `group_call_host` designation, granting them administrative privileges.

🔗 The link from the address bar can be easily shared with candidates or regular users.

👤 When a candidate opens the shared link, they join the meeting as a regular user, devoid of administrative privileges.

📸 For every regular user, the Meet component takes charge of capturing and sending screenshots of their video to our Python server (more details on this in the next section).

By seamlessly managing the different roles and responsibilities within our meetings, the Meet component ensures a smooth and productive collaboration experience. 🚀✨
```jsx

import { useState, useEffect, useRef } from 'react';
import { DyteMeeting, provideDyteDesignSystem } from '@dytesdk/react-ui-kit';
import { useDyteClient } from '@dytesdk/react-web-core';
import Proctor from './Proctor';


// Constants
let LAST_BACKEND_PING_TIME = 0;
const DETECT_FACES_ENDPOINT = 'http://localhost:8000/detect_faces';
const TIME_BETWEEN_BACKEND_PINGS = 30000;

const Heading = ({ text }) => {
    return (
        <div style={{ padding: "10px", textAlign: "center", backgroundColor: "#000", borderBottom: "solid 0.5px gray", height: "3vh" }}><span  className='heading-proctor'>{text}</span></div>
    )
}

const Meet = () => {
    const meetingEl = useRef();
    const [meeting, initMeeting] = useDyteClient();
    const [userToken, setUserToken] = useState();
    const [isAdminBool, setAdminBool] = useState(null);
    const meetingId = window.location.pathname.split('/')[2]

    const joinMeeting = async (id) => {
        const resp = await fetch(`http://localhost:8000/meetings/${id}/participants`, {
            method: "POST",
            body: JSON.stringify({ name: "new user", preset_name: "group_call_host", meeting_id: meetingId }),
            headers: { "Content-Type": "application/json" }
        })
        const respJson = await resp.json()
        console.log(respJson.detail)
        const data = JSON.parse(respJson.detail)
        return data.data.token;
    }

    const isAdmin = async (id) => {
        const resp = await fetch(`http://localhost:8000/is_admin`, {
            method: "POST",
            body: JSON.stringify({ admin_id: window.localStorage.getItem("adminId"), meeting_id: meetingId }),
            headers: { "Content-Type": "application/json" }
        })
        const respJson = await resp.json()
        console.log(respJson)
        setAdminBool(respJson.admin)
    }

    function SendImageToBackendMiddleware() {
        return async (canvas, ctx) => {
            const currentTime = Date.now();            
            if (currentTime - LAST_BACKEND_PING_TIME > TIME_BETWEEN_BACKEND_PINGS) {
                LAST_BACKEND_PING_TIME = currentTime;
                const imgBase64String = canvas.toDataURL('image/png');
                const response = await fetch(DETECT_FACES_ENDPOINT, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({
                        base64_img: imgBase64String,
                        participant_id: meeting?.self.id,
                        participant_name: meeting?.self.name,
                        meeting_id: meetingId
                    }),
                });
                const res = await response.json();
                if (res['multiple_detected']) {
                    console.log('Warning: Multiple faces detected!');
                    sendNotification({
                        id: 'multiple_faces_detected',
                        message: 'Warning: Multiple faces detected!',
                    });
                }
            }
        };
    }

    const joinMeetingId = async () => {
        if (meetingId) {
            const authToken = await joinMeeting(meetingId)
            await initMeeting({
                authToken,
            });
            setUserToken(authToken)
        }
    }

    useEffect(() => {
        if (meetingId && !userToken) joinMeetingId()
        isAdmin()
    }, [])

    useEffect(() => {
        if (userToken) {
            provideDyteDesignSystem(meetingEl.current, {
                theme: 'dark'
            });
        }
    }, [userToken])
    
    useEffect(() => {
        if(isAdminBool === false && meeting?.self) {
            console.log(isAdminBool, "isadmin")
            meeting.self.addVideoMiddleware(SendImageToBackendMiddleware);
        }
    }, [isAdminBool])

    return (
        <div style={{ height: "96vh", width: "100vw", display: "flex" }}>
            { userToken && 
                <>
                    {isAdminBool && <div style={{ width: "40vw", height: "100vh", overflowY: "scroll", backgroundColor: "black", borderRight: "solid 0.5px gray" }}><Heading text={"Proctoring Information"} /><Proctor meeting={meeting} /></div>}
                    {isAdminBool ? <div style={{ width: "60vw", height: "96vh" }}><Heading text={"Proctoring Admin Interface"} /><DyteMeeting mode='fill' meeting={meeting} ref={meetingEl} /></div> : <div style={{ width: "100vw", height: "96vh" }}><Heading text={"Proctoring Candidate Interface"} /><DyteMeeting mode='fill' meeting={meeting} ref={meetingEl} /></div>}
                </>
            }
        </div>
    )
}

export default Meet
```

The `isAdmin` function checks if the current client is an admin by communicating with the Python server. 

The `joinMeeting` function adds the current client to the meeting.

The `SendImageToBackendMiddleware` function sends screenshots of the candidate's video to the Python server.

**Proctor Component**
🔍The Proctor component is designed for admins and is responsible for displaying the list of suspicious candidates in a chat-like format. To use the Proctor component, create a file named `Proctor.jsx` in the `frontend/src` directory .✍️💻

```jsx
import { useEffect, useState } from "react";

const Proctor = ({ meeting }) => {
    const [candidateStatuses, updateCandidateStatusState] = useState([])

    const getCandidateStatus = async () => {
        const response = await fetch("http://localhost:8000/multiple_faces_list", {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                meeting_id: window.location.pathname.split('/')[2],
                admin_id: window.localStorage.getItem("adminId") || "undefined"
            })
        });
        const res = await response.json()
        if(res.details) return undefined
        console.log(res)
        return res
    }

    const updateCandidateStatus = async () => {
        try {
            const res = await getCandidateStatus()
            updateCandidateStatusState(res)
        } catch(e) {
            setError("User don't have admin privileges.")
        }
    }

    useEffect(() => {
        updateCandidateStatus()
    }, [])

    useEffect(() => {
        if(candidateStatuses?.map) {
            const id = setInterval(() => {
                updateCandidateStatus()
            }, 20000)
            return () => {
                clearInterval(id)
            }
        }
    }, [candidateStatuses])

    return(
        <>
            <div style={{ padding: "0px 20px" }}>
                {candidateStatuses?.map && candidateStatuses ? candidateStatuses.map(status => 
                    <div style={{ display: "flex", justifyContent: "start", margin: "50px 20px" }}>
                        <div style={{ marginRight: "20px"}}>
                            <img src="https://images.yourstory.com/cs/images/companies/Dyte-1608553297314.jpg" style={{ borderRadius: "50px", height: "60px", border: "1px double lightblue" }} />
                        </div>
                        <div style={{ textAlign: "center", padding: "10px", backgroundColor: "#2160fd", fontSize: "x-large", fontWeight: "bold", borderRadius: "10px 10px 10px 10px", width: "50vw",  }} >
                            <div style={{ color: "white", padding: "20px 0px" }}>{status[4]}</div>
                            <img style={{ borderRadius: "10px" }} src={status[3]} />
                        </div>
                    </div>) : <div style={{ color: "white" }}>Wait or check if you have admin privileges to access the proctoring dashboard.</div>}
            </div>   
        </>
    )
}

export default Proctor;
```
To start the React app on the local server, run the command `yarn start`. Then, visit `http://localhost:3000/` in your browser to view the Dyte meeting.
<img src="https://user-images.githubusercontent.com/42088801/239780692-37a4f1a7-3901-4e7e-a690-7804dda91972.png">

## Step 2: Setting up the backend 🛠️💻

Now, go back to the root directory of our project and create a new directory called  `backend`  using the following command:
```shell
cd ..
mkdir backend
cd backend
```
Next, let's create a new file called `imagur.py` in the `backend` directory and add the provided code. This code will enable us to upload our screenshots to Imgur.
```jsx
import base64
from fastapi import FastAPI, UploadFile, HTTPException
from httpx import AsyncClient
from dotenv import load_dotenv

load_dotenv()

app = FastAPI()
IMGUR_CLIENT_ID = os.getenv("IMGUR_CLIENT_ID")

async def upload_image(img_data):
    headers = {
        "Authorization": f"Client-ID {IMGUR_CLIENT_ID}"
    }
    data = {
        "image": img_data
    }
    
    async with AsyncClient() as client:
        response = await client.post("https://api.imgur.com/3/image", headers=headers, data=data)
        
    if response.status_code != 200:
        raise HTTPException(status_code=500, detail="Could not upload image.")

    print(response.json())
    return response.json()["data"]["link"]
```
Now, let's create a new file called `app.py` and add the provided code to it. This code will be used to implement the backend functionality for our application.
   
```jsx
import base64
import io
import logging
import random


import uvicorn
from fastapi import FastAPI, File, UploadFile
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from imgur import upload_image
import face_recognition
import psycopg2
import json

import os
import base64
from fastapi import FastAPI, HTTPException
from starlette.responses import RedirectResponse
from pydantic import BaseModel
from typing import Optional
from dotenv import load_dotenv
from httpx import AsyncClient
import uuid

load_dotenv()

DYTE_API_KEY = os.getenv("DYTE_API_KEY")
DYTE_ORG_ID = os.getenv("DYTE_ORG_ID")

API_HASH = base64.b64encode(f"{DYTE_ORG_ID}:{DYTE_API_KEY}".encode('utf-8')).decode('utf-8')

DYTE_API = AsyncClient(base_url='https://api.cluster.dyte.in/v2', headers={'Authorization': f"Basic {API_HASH}"})

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)
# save logs to a file
fh = logging.FileHandler("app.log")
fh.setLevel(logging.DEBUG)
# create formatter
formatter = logging.Formatter("%(asctime)s - %(name)s - %(levelname)s - %(message)s")
# add formatter to fh
fh.setFormatter(formatter)
# add fh to logger
logger.addHandler(fh)


class ParticipantScreen(BaseModel):
    base64_img: str
    participant_id: str
    meeting_id: str
    participant_name: str

class ProctorPayload(BaseModel):
    meeting_id: str
    admin_id: str

class AdminProp(BaseModel):
    meeting_id: str
    admin_id: str

origins = [
    # allow all
    "*",
]

app = FastAPI()

# enable cors
app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],  # allow all
    allow_headers=["*"],  # allow all
)


@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.post("/is_admin/")
async def multiple_faces_list(admin: AdminProp):
    conn = psycopg2.connect(
            dbname=os.getenv('DB_USER'), 
            user=os.getenv('DB_USER'), 
            password=os.getenv('DB_PASSWORD'),
            host=os.getenv('DB_HOST'),
            port=5432
    )
    cur = conn.cursor()
    cur.execute("SELECT count(1) FROM meeting_host_info WHERE meeting_id = %s AND admin_id = %s", (admin.meeting_id, admin.admin_id,))
    
    count = cur.fetchone()[0]
    
    if(count > 0):
        return { "admin": True }
    else:
        return { "admin": False }




@app.post("/multiple_faces_list/")
async def multiple_faces_list(meeting: ProctorPayload):
    conn = psycopg2.connect(
            dbname=os.getenv('DB_USER'), 
            user=os.getenv('DB_USER'), 
            password=os.getenv('DB_PASSWORD'),
            host=os.getenv('DB_HOST'),
            port=5432
    )
    cur = conn.cursor()
    cur.execute("SELECT count(1) FROM meeting_host_info WHERE meeting_id = %s AND admin_id = %s", (meeting.meeting_id, meeting.admin_id,))
    
    count = cur.fetchone()[0]
    
    if(count > 0):
        cur.execute("SELECT * FROM meeting_proc_details WHERE meeting_id = %s ORDER BY ts DESC", (meeting.meeting_id,))
        rows = cur.fetchall()
        conn.commit()
        cur.close()
        conn.close()
        return rows
    else:
        conn.commit()
        cur.close()
        conn.close()
        raise HTTPException(status_code=401, detail="Participant dose not has admin role")


    


@app.post("/detect_faces/")
async def detect_faces(participant: ParticipantScreen):
    img_data = participant.base64_img.split(",")[1]
    img_data_dc = base64.b64decode(participant.base64_img.split(",")[1])

    file_obj = io.BytesIO(img_data_dc)
    img = face_recognition.load_image_file(file_obj)

    face_locations = face_recognition.face_locations(img)

    if len(face_locations) > 1:
        logger.info(
            f"Detected more than one face for participant {participant.participant_id}"
        )

        upload_resp = await upload_image(img_data)

        conn = psycopg2.connect(
            dbname=os.getenv('DB_USER'), 
            user=os.getenv('DB_USER'), 
            password=os.getenv('DB_PASSWORD'),
            host=os.getenv('DB_HOST'),
            port=5432
    )
        cur = conn.cursor()

        cur.execute("CREATE TABLE IF NOT EXISTS meeting_proc_details (ts TIMESTAMP, meeting_id VARCHAR(255), participant_id VARCHAR(255), img_url VARCHAR(255), verdict VARCHAR(255))")

        verdict = f"Detected more than one face for participant \"{participant.participant_name}\" and participant_id {participant.participant_id}"
        cur.execute("INSERT INTO meeting_proc_details (ts, meeting_id, participant_id, img_url, verdict) VALUES (current_timestamp, %s, %s, %s, %s)", (participant.meeting_id, participant.participant_id, upload_resp, verdict))

        conn.commit()
        cur.close()
        conn.close()


        return { "id": participant.participant_id, "multiple_detected": True, "url": upload_resp }

    return {"id": participant.participant_id, "multiple_detected": False}


class Meeting(BaseModel):
    title: str


class Participant(BaseModel):
    name: str
    preset_name: str
    meeting_id: str


@app.post("/meetings")
async def create_meeting(meeting: Meeting):
    response = await DYTE_API.post('/meetings', json=meeting.dict())
    if response.status_code >= 300:
        raise HTTPException(status_code=response.status_code, detail=response.text)
    admin_id = ''.join(random.choices('abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789', k=32))
    resp_json = response.json()
    resp_json['admin_id'] = admin_id
    meeting_id = resp_json['data']['id']

    conn = psycopg2.connect(
            dbname=os.getenv('DB_USER'), 
            user=os.getenv('DB_USER'), 
            password=os.getenv('DB_PASSWORD'),
            host=os.getenv('DB_HOST'),
            port=5432
    )
    cur = conn.cursor()
    cur.execute("INSERT INTO meeting_host_info (ts, meeting_id, admin_id) VALUES (CURRENT_TIMESTAMP, %s, %s)", (meeting_id, admin_id))
    conn.commit()
    cur.close()
    conn.close()

    return resp_json


@app.post("/meetings/{meetingId}/participants")
async def add_participant(meetingId: str, participant: Participant):
    client_specific_id = f"react-samples::{participant.name.replace(' ', '-')}-{str(uuid.uuid4())[0:7]}"
    payload = participant.dict()
    payload.update({"client_specific_id": client_specific_id})
    del payload['meeting_id']
    resp = await DYTE_API.post(f'/meetings/{meetingId}/participants', json=payload)
    if resp.status_code > 200:
        raise HTTPException(status_code=resp.status_code, detail=resp.text)
    return resp.text

if __name__ == "__main__":
    uvicorn.run("app:app", host="localhost", port=8000, log_level="debug", reload=True)

```
The provided code sets up a 🏃‍♂️FastAPI application with a specific endpoint called `/detect_faces`. This endpoint is responsible for receiving a base64 encoded image as input. The code then utilizes the😄 `face_recognition` library to detect faces within the image.

To set up the backend environment 🌍 and specify the necessary dependencies, follow these steps:

1. Create a new file called `.env` inside the backend directory.
2. Open the `.env` file and add the following environment variables with their corresponding values:
```
DYTE_ORG_ID=<ID>
DYTE_API_KEY=<KEY>
IMGUR_CLIENT_ID=<ID>
DB_USER=<ID>
DB_PASSWORD=<PASSWORD>
DB_HOST=<HOST>
```
Replace `<ID>`, `<KEY>`, `<PASSWORD>`, and `<HOST>` with the actual values relevant to your setup.

3. Create a new file named `requirements.txt` in the backend directory.
4. Open the `requirements.txt` file and add the following dependencies:
```
fastapi
uvicorn
face_recognition
numpy
python-multipart
py-dotenv
```

These files, `.env` and `requirements.txt`, will help manage the necessary environment variables and dependencies for your backend application.

Next, we'll set up a virtual environment for our project and install the necessary dependencies. Run the following commands to get started:

```bash
python -m venv venv
source venv/bin/activate # for linux/mac
venv\Scripts\activate.bat # for windows
pip install -r requirements.txt
```
 Now, we can start the backend server using the following command:

```bash
uvicorn app:app --reload --port 8000
```
This Python server serves multiple purposes, including creating and joining meetings, detecting multiple faces, and retrieving a list of suspicious candidates. When we access the `/detect_faces` endpoint with an image file encoded as a base64 string, the `multiple_detected` key in the response will be set to `True` if there are more than one faces in the image; otherwise, it will be set to `False`. This functionality can be utilized in our frontend by utilizing the participant's webcam feed to detect whether there are multiple faces in the frame.

## Step 3: Adding the face detection logic to the frontend 📷🔍
Now that we have a powerful backend server capable of detecting faces, we can incorporate the face detection logic into our frontend. To do this, we'll start by adding a few constants to the `frontend/src/App.tsx` file that we worked on earlier. These constants will help us with the integration process.
We will be using the above constants in the `SendImageToBackendMiddleware` function which we will add to our `App` component, just after the `useDyteClient` hook.

The `SendImageToBackendMiddleware` is a special feature provided by Dyte Video. It acts like an add-on that allows us to easily apply effects and filters to the audio and video streams. In our case, we use this middleware to process and send the participant's webcam feed to the backend server for face detection. It's a convenient way to enhance the functionality of our video conferencing application.

Here, we're using the middleware feature to capture the participant's webcam feed as a canvas object. We convert this canvas image into a base64-encoded format and send it to our backend server. To maintain server efficiency, we limit backend pings to every 30 seconds.

If the backend server detects multiple faces in the image, it returns a value of True for the `multiple_detected` key. In such cases, we use the `sendNotification` function to alert the participant with a notification. This ensures that participants are notified when multiple faces are detected during the meeting.

The middleware code is as follows:

```jsx
...
    const [meeting, initMeeting] = useDyteClient();

    async function SendImageToBackendMiddleware() {
        return async (canvas: HTMLCanvasElement, ctx: CanvasRenderingContext2D) => {
            const currentTime = Date.now();
            if (currentTime - LAST_BACKEND_PING_TIME > TIME_BETWEEN_BACKEND_PINGS) {
                LAST_BACKEND_PING_TIME = currentTime;
                const imgBase64String = canvas.toDataURL('image/png');
                const response = await fetch(DETECT_FACES_ENDPOINT, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({
                        base64_img: imgBase64String,
                        participant_id: meeting?.self.id,
                    }),
                });
                const res = await response.json();
                if (res['multiple_detected']) {
                    console.log('Warning: Multiple faces detected!');
                    sendNotification({
                        id: 'multiple_faces_detected',
                        message: 'Warning: Multiple faces detected!',
                    });
                }
            }
        };
    }
...
```
## Step 4: Alerting the participant when multiple faces are detected
By incorporating the code we discussed, we have successfully added a basic proctoring functionality to our Dyte meeting. 🎉 Every 30 seconds, the app captures a screenshot of the participant's webcam feed 📸 and sends it to the backend server. The backend then detects the number of faces in the image 👥. If it detects more than one face, a warning notification is sent to the participant . Additionally, the backend logs the participant's ID and the time of detection , providing valuable information for identifying potential cheating incidents during the meeting. This enables us to track and review participant behavior effectively .

## Step 5: Testing the Proctoring System

Ta-da! 🎩✨ It's time to put our proctoring system to the test and see it in action!

<img src="https://github.com/ankiiitraj/dyting/blob/main/dyte-admin-ss.png?raw=true">
<img src="https://github.com/ankiiitraj/dyting/blob/main/dyte-candidate-ss.png?raw=true">

## Conclusion🎉

Celebrate! 🎉✨ We've built a powerful proctoring system with Dyte, ensuring integrity and fairness in online exams and interviews. But that's not all! We can now create our own customized online classroom or meeting platform.

Picture hosting virtual classrooms, collaborative sessions, or team meetings, all with the added security of proctoring. The possibilities are endless!

Unleash your creativity and revolutionize your online interactions with Dyte. Let's build collaborative applications and make a lasting impact in the digital world. 🚀💪
