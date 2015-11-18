## Hello Wifi Blink!

In this tutorial we will configure our NodeMCU board as an Access Point so we can connect to it through a web browser.

We will make a simple page that sends requests to our server and toggles one of the board's LED. In order to do that we will introduce use `net` module to create a local HTTP server.

You can check the full API in the NodeMCU [wiki][net-module].

### HTTP Local Server

Next, we are going to use the `net` module to create a local HTTP server. If you have worked with nodejs before this will feel somehow familiar, as it is equally as straight forward.

```lua
srv:listen(80, function(conn)
...
end)
```

The `net.createServer` function takes a callback that gets the current connection instance as an argument. We register a listener for the **receive** event, which get's triggered every time we make a request to the HTTP server.

```lua
conn:on('receive', function(conn, payload)
...
end)
```

We are going to handle three routes:

* GET /
* POST /led/on
* POST /led/off

Our index page will be really simple, with a button to toggle the current state of the LED.

The button has a small bit of [hacky] JavaScript to handle the HTTP request to the server:

```javascript
x = new XMLHttpRequest();
x.open('POST', 'led/'+(b?'on':'off'));
x.send();
b=!b;
```

The POST request to the **led** endpoint are self explanatory, they effectively turn the LED on and off.


Once we are connected to the **WEE_THINGS_XXXXXX** local network, this will enable us to go connect to the board using a browser, at the address [http://192.168.4.1][localhost].


![wifi-blink](https://raw.githubusercontent.com/goliatone/wee-things-workshop/master/images/hello-wifi-blink-001.png)


**NOTE:**

If you upload the script file to the board, make changes locally and then re-upload the file, most likely you will get the following error:

>boot.lua:34: only one tcp server allowed

To prevent this error, we first disconect any previous server instance.

```lua
if server then srv:close() end
```

### Source Code

The full script:

`boot.lua`:
```lua
-- Initialize WiFi access point
wifi.setmode(wifi.SOFTAP)

-- Configure the access point
cfg = {
    ssid = "WEE_THINGS_"..node.chipid(),
    pwd = "weething"
}
wifi.ap.config(cfg)

print('Wifi mode: ', wifi.getmode())
print('MAC: ', wifi.sta.getmac())
print('')
print('Created WiFi network:')
print('Network Name')
print('- SSID: ', cfg.ssid)
print('Security: WPA/WPA 2 Personal')
print('- PWD: ', cfg.pwd)
print('')

address = {
  ip = '192.168.4.1',
  netmask = '255.255.255.0',
  gateway = '192.168.4.1'
}
wifi.ap.setip(address)

print('IP: ', wifi.ap.getip())

-- Initialize LED pin
gpio.mode(4, gpio.OUTPUT)

-- Close any server instances running
if server then srv:close() end

-- Create a server
srv = net.createServer(net.TCP)

srv:listen(80, function(conn)
    -- close the connection once we are done
    conn:on('sent', function(conn)
        conn:close()
    end)

    -- handle all incomming requests
    conn:on('receive', function(conn, payload)
        -- Handle requests to http://192.168.4.1
        if payload:find('GET /') == 1 then
            conn:send("HTTP/1.0 200 OK\r\nContent-type: text/html\r\nServer: wee-thing\r\n\n")
            conn:send('<html><head><meta charset="utf-8"><title>Wee Things</title></head>')
            conn:send('<style>html{background-color:#212121; color:#fafafa}</style>')
            conn:send('<body><h3>Wee Things</h3><p>Use the button to toggle the board LED</p>')
            conn:send('<input type="button" value="Toggle LED" onclick="x=new XMLHttpRequest();x.open(\'POST\', \'led/\'+(b?\'on\':\'off\'));x.send();b=!b;" /></body>')
            conn:send('<script>b=true</script></html>')
        end

        -- POST request to "http://192.168.4.1/led/on"
        if payload:find('POST /led/on') == 1 then
            gpio.write(4, gpio.LOW) -- turn LED on
            conn:send('HTTP/1.0 204 No Content\r\n\r\n')
        end

        -- POST request to "http://192.168.4.1/led/off"
        if payload:find('POST /led/off') == 1 then
            gpio.write(4, gpio.HIGH) -- turn LED off
            conn:send('HTTP/1.0 204 No Content\r\n\r\n')
        end

        conn:close()
    end)
end)
```

It is available as a gist [here][gist].



**NOTE:**
The source code in this example was taken from [here][tut] and modified where necessary.


[gist]: https://gist.github.com/goliatone/001a3b147ddfee3e8451
[tut]: https://github.com/Densaugeo/ESP-8266-NodeMCU-Tutorial
[wiki]: https://github.com/nodemcu/nodemcu-firmware/wiki/nodemcu_api_en
[localhost]: http://192.168.4.1
[net-module]: https://github.com/nodemcu/nodemcu-firmware/wiki/nodemcu_api_en#net-module