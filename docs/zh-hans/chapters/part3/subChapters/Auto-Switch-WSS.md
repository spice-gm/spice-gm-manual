### Spice Auto Switch Wss 改造


```html
<label for="if_websocket_secure">Protocol:</label> <select id="if_websocket_secure"> </select> <br>
```
```js
... ...
if (window.location.protocol == 'https:') {
    document.getElementById("if_websocket_secure").append(new Option("wss", "wss://"));
} else {
    document.getElementById("if_websocket_secure").append(new Option("ws", "ws://"));
}
... ...
function connect() {
  var host, port, password, scheme = "ws://", uri;

  host = document.getElementById("host").value;
  port = document.getElementById("port").value;
  password = document.getElementById("password").value;
  var select_websocket_scheme = document.getElementById("if_websocket_secure");
  scheme = select_websocket_scheme.options[select_websocket_scheme.selectedIndex].value;
  if ((!host) || (!port)) {
      console.log("must set host and port");
      return;
  }

  if (sc) {
      sc.stop();
  }

  uri = scheme + host + ":" + port;

  document.getElementById('connectButton').innerHTML = "Stop Connection";
  document.getElementById('connectButton').onclick = disconnect;

  try {
      sc = new SpiceHtml5.SpiceMainConn({
          uri: uri, screen_id: "spice-screen", dump_id: "debug-div",
          message_id: "message-div", password: password, onerror: spice_error, onagent: agent_connected
      });
  }
  catch (e) {
      alert(e.toString());
      disconnect();
  }
}
```
