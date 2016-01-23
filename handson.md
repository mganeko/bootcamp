# bootcamp handson

WebRTC Conference 2016 で行われる WebRTC Boot Camp のHandsonの手順です

# Part 1. カメラを使ってみよう

## navigator.getUserMedia()
* カメラの映像/マイクの音声を取得できる
* ベンダーブレフィックスが付いている
 * Chrome: navigator.webkitGetUserMedia()
 * Firefox: navigator.mozGetUserMedia()
* 最新の仕様では、navigator.MediaDevices.getUserMedia()
 * 今日は使いません
* ユーザに許可を求めるダイアログが表示されます
* 取得に成功するとMediaStream オブジェクトが得られます
* ※Chrome 47からは https://〜でしか利用できなくなりました
 * http://localhost は例外として利用できます

````javascript
// プレフィックスを吸収する
navigator.getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia;

// カメラの映像を取得する場合
navigator.getUserMedia( {video: true},
 function(stream) { // 成功時のコールバック
 },
 function(err) { // エラー時のコールバック
 }
);   
````
### マイクの音声も取得する場合
* navigator.getUserMedia( {video: true, audio: true}, 
* ※会場でやるとハウリングしてうるさいので、今日は音声無しで

## 映像をvideo要素で再生
* window.URL.createObjectURL(stream) でURLを取得
 * blob://〜 という表記のURL
* video要素のsrc に指定して再生

````javascript
video.src = window.URL.createObjectURL(stream);
video.play();
````
## 実際にやってみよう(1)(2)

* media_0.html をエディタで開いてください
* startCamera() 関数の (1), (2) を記述してみてください
* [start]ボタンを押して、試してみましょう

## 解答例
````javascript
  function startCamera() {
    // (1) getUserMediaを使って、cameraからメディアストリームを取得してください
    //     また、それをlocalStreamにセットしておいてください
    navigator.getUserMedia( {video : true},
      function(stream) { // success
        localStream = stream;

        // (2) それを localVideo に表示してください
        localVideo.src = window.URL.createObjectURL(localStream);
        localVideo.play();
      },
      function(err) { // error
        console.err('getUserMedia error', err);
      }
    );
  }
````


## 映像の停止
* video.pause()
* window.URL.revokeObjectURL(url) でURLを解放
* ストリームを停止
 * MediaStream.stop() はChrome 47から使えなくなりました
 * MediaTrack.stop() を使う必要があります
 
## 実際にやってみよう (3)(4)
* 先ほどの続きで、stopCamera() 関数の (3), (4) を記述してみてくだい
* MediaStreamを停止させるための関数を stopStream(stream) として用意していますので、ご利用ください
* [start]→[stop]ボタンを押して、試してみましょう

## 解答例
````javascript
  function stopCamera() {
    // (3) localVideoの再生を停止させてください
    localVideo.pause();
    window.URL.revokeObjectURL(localVideo.src);
    localVideo.src = ''; // Firefox WARN
    
    // (4) メディアストリームを停止させてください
    // ※注意：mediaStream.stop()は、Chrome 47から使えなくなりました
    // localStream.stop(); //Chrome 47 NG, Firefox 43 OK
    stopStream(localStream);
    localStream = null;   
  }
````

## 参考：新しい getUserMedia
* [Media Capture and Streams](http://www.w3.org/TR/mediacapture-streams/) では、新しいAPIが定義されています
* navigator.MediaDevices.getUserMedia()
* Promiseを返します
````javascript
navigator.mediaDevices.getUserMedia(
  {video: true}
).then(function (stream) {
  // 成功時の処理
}).catch(function (reason) {
   // 例外時の処理
});
````


# Part2. PeerConnectionで通信してみよう

## RTCPeerConnection
* WebRTCで通信するためには、RTCPeerConnetionを使用する
* ベンダーブレフィックスが付いている
 * Chrome: window.webkitRTCPeerConnetion
 * Firefox: window.mozRTCPeerConnection
* Peer-to-Peer 通信を確立するために、2種類の情報を交換する
 * SDP (Session Description Protocol)
 * ICE Candidate (Interactive Connectivity Establishment Candidate)
* SDP ... 通信するPeerの能力や、条件について
 * 映像、音声、データーの有無
 * 使いたいコーデック、使いたい帯域幅
 * 通信を始める側： Offer
 * 通信に応答する側： Answer
 など
* ICE Candidate
 * 通信に使う経路の情報

## 注意
* ICE Candidate を収集するには、デバイスがネットワークに繋がっている必要がある
 * ネットワーク無し、localhostのみだとICE candidate が収集できない
* テザリング等でデバイスをネットワークに接続してから、実行してください
 * 外部と実際には通信しなくても、外部と接続できるネットワークが必要です


## シグナリング
* 最初は通信相手のことをお互い知らない
* 情報をなんらかの方法で交換する → シグナリング
* 仲介役となるサーバー → シグナリングサーバー
* 今回は、サーバーは「あなた」→ 手動でシグナリング

## デモ

## SDP交換の概略

### Offer側
* Offer SDP 生成
* 自分のSDPを覚える
 * RTCPeerConnection.setLocalDescription()
* 相手に送る
* 相手から Answer SDP を受け取ったら、それを覚える
 * RTCPeerConnection.setRemoteDescription()
 
※非同期処理になるので、コールバックを使って記述する

### Answer側
* 相手から Offer SDP を受け取ったら、それを覚える
 * RTCPeerConnection.setRemoteDescription()
* Answer SDP を生成
* 自分のSDPを覚える
 * RTCPeerConnection.setLocalDescription()
* 相手に送り返す
 
※非同期処理になるので、コールバックを使って記述する

````javascript
// プレフィックスを吸収する
RTCPeerConnection = window.RTCPeerConnection || window.webkitRTCPeerConnection || window.mozRTCPeerConnection;

// PeerConnectionオブジェクトを生成する
var peerConnection = new RTCPeerConnection({"iceServers":[]});
  // iceServersは NAT/Firewall越えを行う場合に指定
  // 今回は指定は空っぽでOK
  
// Offer SDPを生成する
peerConnection.createOffer(
  function(sessionDescription) {
    // 成功した場合
  },
  function(err) {
    // 失敗した場合
  },
  {} // オプション指定
);

// Answer SDPを生成する
peerConnection.createAnswer(
  function(sessionDescription) {
    // 成功した場合
  },
  function(err) {
    // 失敗した場合
  },
  {} // オプション指定
);


// SDP を受け取った場合
peerConnection.setRemoteDescription(sessionDescription,
  function() {
    // 成功
  },
  function(err) {
    // 失敗
  }
);

````

## ICE Candidateの交換
* ICE Candidate (通信経路の候補)は複数個ある
* 非同期に順々に収集される
 * RTCPeerConnection.onicecandidate() 
* 早く見つかったものから随時交換するのが Trikle ICE
 * 早く繋がるケースが多い
* 全て見つけてから、まとめて交換するのが Vanilla ICE
 * 処理はシンプル。※今日はこちらを利用
 
````javascript
peerConection.onicecandidate = function (evt) {
  if (evt.candidate) {
    // 個々のcandidateを収集をした時の処理
    // Trikle ICE の場合、ここで処理する
  } else {
    // 全てのICE candidateが収集し終わったタイミング
    // Vanilla ICE の場合、ここで処理する
    var sdpWithICE = peerConnection.localDescription.sdp;
  }
};

````

## 手動シグナリングをやってみよう(1)〜(7)
* hand_vanilla_0.html をエディタで開いてください
* makeOffer()関数の(1),(2)を記述してみてください
* setOfferText()関数の(3),(4)を記述してみてください
* makeAnswer()関数の(5),(6)を記述してみてください
* setAnswerText()関数の(7)を記述してみてください

## 解答例
````javascript
  // Offerを生成する
  function makeOffer() {
    // (1) createOfferを呼び出して、Offer SDPを生成する
    peerConnection.createOffer(function (sessionDescription) { // in case of success
      // (2) 生成されたSDPを、peerConnetionにセットする
      //    ※Vanilla ICEを用いるため、すぐにはOfferを相手に送信しない
      peerConnection.setLocalDescription(sessionDescription,
        function() {
          console.log('setLocalDescription() succsess');
          
          // ※Trickle ICEの場合は、このタイミングでOffer SDPを相手に送信する
          // ※Vanilla ICEの場合には、まだ送らない
        },
        function(err) {
          console.error('setLocalDescription() ERROR: ', err);
        }
      );
      console.log('-- Create Offer SDP --');
      console.log(sessionDescription);
    }, function (err) { // in case of error
      console.error('Create Offer failed:', err);
    }, mediaConstraints);
  }
````

````javascript
  // Offerを受け取る
  function setOfferText(text) {
    // SDPの文字列からRTCSessionDescriptionのオブジェクトを生成
    var offer = new RTCSessionDescription({
      type : 'offer',
      sdp : text,
    });
    
    // (3) 生成したオブジェクトを、peerConnetionにセットする
    // 相手側のSDPをセットする
    peerConnection.setRemoteDescription(offer,
      function() {
        console.log('setRemoteDescription(offer) succsess');
        
        // (4) さらに、適切なタイミングで用意している関数 makeAnswer() を呼び出す
        // コールバックが呼ばれたら、Answerを生成する
        makeAnswer();
      },
      function(err) {
        console.error('setRemoteDescription(offer) ERROR: ', err);
      }
    );
  }
````

````javascript
  // Answer を生成する
  function makeAnswer() {
    // (5) createAnswerを呼び出して、Answer SDPを生成する
    peerConnection.createAnswer(function (sessionDescription) { // in case of success
      // (6) 生成されたSDPを、peerConnetionにセットする
      //    ※Vanilla ICEを用いるため、すぐにはOfferを相手に送信しない
      peerConnection.setLocalDescription(sessionDescription,
        function() {
          console.log('setLocalDescription() succsess');
          // ※Trickle ICEの場合は、このタイミングでAnswer SDPを相手に送信する
          // ※Vanilla ICEの場合には、まだ送らない
        },
        function(err) {
          console.error('setLocalDescription() ERROR: ', err);
        }
      );

      console.log('-- Create Answer SDP --');
      console.log(sessionDescription);
    }, function (err) { // in case of error
      console.error('Create Answer failed:', err);
    }, mediaConstraints);
  }
````

````javascript
  // Answerを受け取る
  function setAnswerText(text) {
    // SDPの文字列からRTCSessionDescriptionのオブジェクトを生成
    var answer = new RTCSessionDescription({
      type : 'answer',
      sdp : text,
    });
   
    // (7) 生成したオブジェクトを、peerConnetionにセットする
    // 相手側のSDPをセットする
    peerConnection.setRemoteDescription(answer,
      function() {
        console.log('setRemoteDescription(answer) succsess');
      },
      function(err) {
        console.error('setRemoteDescription(answer) ERROR: ', err);
      }
    );
  }
````



 


 




