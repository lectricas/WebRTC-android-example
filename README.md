# Android WebRTC Demo

I summarized the knowledge I learned while using WebRTC in Android apps. Afterwards, based on this content, a demo app was created to develop a simple video communication service between two Android smartphones.

ℹ️See the [WebRTC](http://webrtc.github.io/webrtc-org/) webpage for release notes and issue tracking.

## 0. Demo Video.
[![Video Label](http://img.youtube.com/vi/VQBzslcEH4o/0.jpg)](https://youtu.be/VQBzslcEH4o?t=0s)

## 1. Summary
1. After exchanging offers / answers with the other party and exchanging Ice Candidates, the two of them conduct P2P direct communication.
2. Process (1) is done through [socket communication.
3. A simple image showing the socket communication that occurs while using WebRTC is attached.
![webrtc-socket-img](https://github.com/skfo763/WebRTC-android-example/blob/master/.github/art/webrtc-socket-img.png)
## 2. Architecture

### Overview
1. This demo app uses the pre-built library provided by [WebRTC Guidelines](http://webrtc.github.io/webrtc-org/native-code/android/).
2. Inside the RTC module, the Manager in charge of PeerConnection and the Manager in charge of Socket communication were divided.
3. You can implement video chat directly through FaceChatRtcManager.
4. Server specifications such as signaling server, turn / coturn are not in this repository. Browse other related repo.

### Structure
![image](./.github/art/videochat_structure_brand_new.png)
1. View -> RtcManager communication
   - The demo app was created in a way that ViewModel manages RtcManager.
   - If SurfaceView is registered with RtcManager, RtcManager initializes the view, captures the camera, and renders it.
   - Views call methods in RtcManager according to their lifecycle (*initialize, attach, destroy, etc..*)
   - When creating RtcManager, you must put the constructor of IVideoChatViewModelListener interface.

2. RtcManager -> View communication
   - Deliver RTC events to the view through the IVideoChatViewModelListener interface.
   - Therefore, the class of the view or view model layer must implement IVideoChatViewModelListener.

3. Communication between RtcManager <-> PeerManager
   - Call the method provided by the WebRTC library as it is, without any explicit work.

4. Communication between RtcManager <-> SocketConnect
   - If your service wants to implement custom socket events through a signaling server, create a SocketEventListener or a separate interface to communicate.
   - The demo app implemented a custom event to send and receive data in an event that matches two users.

#### old version (2020.07.10 release)
<details>
<summary>
unfold
</summary>
<p>
![image](./.github/art/android-summary-architecture.png)
1. View <-> Communication between RTC modules
- Pass the SurfaceViewRenderer declared in xml in the Android app module to the RTC module.
- The SurfaceViewRenderer passed in this way configures the RTC's VideoStream and AudioStream.
- In the app module, through [RtcModuleInterface](https://github.com/skfo763/WebRTC-android-example/blob/master/rtc/src/main/java/com/skfo763/rtc/contracts/RtcModuleInterface.kt) Controls the RTC module.
- In the rtc module, [RtcViewInterface](https://github.com/skfo763/WebRTC-android-example/blob/master/rtc/src/main/java/com/skfo763/rtc/contracts /RtcViewInterface.kt) to the app.
2. RTC <-> Communication between Sockets
    - In the case of predefined offer / answer / icecandidate, the rtc library exchanges data on its own without requiring separate socket communication processing.
    - If you want each service to independently issue and receive a specific event during a video call, send and receive through the interface between RTC and socket.
</details>


### Details
#### Property
1. localVideoSource: Manages the creation of video tracks taken by local cameras
~~~
private val localVideoSource by lazy {
peerConnectionFactory.createVideoSource(false)
}
~~~
2. localVideoTrack : Contains the video stream taken by the local camera
~~~
private val localVideoTrack by lazy {
peerConnectionFactory.createVideoTrack(VIDEO_TRACK_ID, localVideoSource)
}
~~~
3. surfaceTextureHelper: Responsible for the texture (texture, color, etc.) of the image captured by the local camera
~~~
surfaceTextureHelper = SurfaceTextureHelper.create(Thread.currentThread().name, rootEglBase.eglBaseContext)
~~~
4. localStream: When I send my audio/video tracks to the other party, I put them in this localStream and send them.
~~~
val localStream: MediaStream by lazy {
peerConnectionFactory.createLocalMediaStream(LOCAL_STREAM_ID)
}
~~~
5. videoCaptureManager: Handles Camera2 / Camera1, front / back facing related processing
~~~
private val videoCaptureManager = VideoCaptureManager.getVideoCapture(context)
~~~
6. peerConnection: Data exchanged with the other party
~~~
protected fun buildPeerConnection(): PeerConnection? {
  val rtcConfig = PeerConnection.RTCConfiguration(iceServer).apply {
  /* TCP candidates are only useful when connecting to a server that supports. ICE-TCP. */
  iceTransportsType = PeerConnection.IceTransportsType.RELAY
        tcpCandidatePolicy = PeerConnection.TcpCandidatePolicy.DISABLED
        bundlePolicy = PeerConnection.BundlePolicy.MAXBUNDLE
        rtcpMuxPolicy = PeerConnection.RtcpMuxPolicy.REQUIRE
        continualGatheringPolicy = PeerConnection.ContinualGatheringPolicy.GATHER_CONTINUALLY
        /* Enable DTLS for normal calls and disable for loopback calls. */
		enableDtlsSrtp = true
		// sdpSemantics = PeerConnection.SdpSemantics.UNIFIED_PLAN
	    /* Use ECDSA encryption. */ // keyType = PeerConnection.KeyType.ECDSA  }

  return peerConnectionFactory.createPeerConnection(rtcConfig, observer) ?: kotlin.run {
	  observer.onPeerError(isCritical = true, showMessage = false, message = PEER_CREATE_ERROR)
	  null
  }
}
~~~
#### PeerConnection connection order
**--- create activity ---**
1. Initialize localStream, localVideoTrack, videoCaptureManager
2. Add localVideoTrack to localStream
3. Initialize surfaceTextureHelper, localVideoSource
4. Calling videoCaptureManager.initialize() method -> Ready to shoot local camera
5. Start shooting local camera

**--- Intro ---**

6. Add localVideoTrack to sv_face_chat_intro_preview -> From this point on, my image is displayed on the intro surface view.

**--- waiting for matching ---**

7. Add localVideoTrack to sv_face_chat_waiting_local -> From this point on, my image is displayed on the surface view of the waiting screen.
8. Initialize the iceServer address value
9. Initialize peerConnection
10. Connect localStream to peerConnection

**--- Enter Busy ---**

11. Add localVideoTrack to sv_face_chat_call_local -> From this point on, my image is displayed on the small surface view of the call screen.

**--- onAddStream() method called: peer connection completed---**

12. Add remoteVideoTrack of the other party to sv_face_chat_call_remote -> From this point on, the other party's appearance is displayed.

**--- End call ---**

13. Close peerConnection, null
14. Depends on exit logic
- When swiping: Proceed again from step 7
- When the Cancel button is pressed and finished: Proceed again from step 6

**--- End of service ---**

15. If peerConnection is alive, remove localStream from it
16. Remove localVideoTrack from localStream
17. End of local camera shooting
18. End of use of localVideoSource
19. videoCaptureManager to null