### Spice Auto Switch Wss 改造

```js
function connect() {
  ... ...
  if (window.location.protocol == 'https:') {
      scheme = "wss://";
  }
  if ((!host) || (!port)) {
      console.log("must set host and port");
      return;
  }
  ... ...
}
```
