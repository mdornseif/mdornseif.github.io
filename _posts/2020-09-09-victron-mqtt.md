# Victron Energy MQTT (en)

Victron Energy products are leading in On-Sore and Off-Shore autonomous power supply for small scale installations. 

The Victron GX product range is "The Brains" of it all, most often represented by the Color Control GX or Cerbo GX. 
With [Venus OS](https://github.com/victronenergy/venus/wiki) they actually run an OpenSource Operating system. 

It offers MQTT but the MQTT needs to be "triggered" by regular publishing of messages from the client. E.g.:

    mosquitto_pub -h <VictronIP> -t 'R/<PortalID>/system/0/Serial' -I myclient_ -m ''

Publishing this keep-alive-messages in client code is somewhat tedius. In Python on POSIX Systems this should do the job:

    import time, signal
    import paho.mqtt.client as mqtt
    
    PORTAL_ID = 'd0ff50ffffff'
    CCGX_ADDR = '192.168.50.3'
    TOPIC = "N/{}/#".format(PORTAL_ID)
    KEEPALIVE = 55
    
    def on_connect(client, userdata, flags, rc):
        """Subscribe to desired Topic."""
        client.subscribe(TOPIC)
    
    # The callback for when a PUBLISH message is received from the server.
    def on_message(client, userdata, msg):
        """Print incomming message."""
        try: # paho.mqtt is hiding errors ...
            # N/<portal ID>/<service_type>/<device instance>/<D-Bus path>
            parts = msg.topic.split('/')
            portal_id, service_type, device_instance = parts[1:3]
            dbus_path = '/'.join(parts[4:])
            print(portal_id, service_type, device_instance, dbus_path, time.time(), msg.payload)
        except Exception as e: 
            print(e)
    
    def keepalive_handler(signum, frame):
        """Publish at an interval something to keep the broker active."""
        # Needed to keep Broker active
        print('keepalive')
        client.publish('R/{}/system/0/Serial'.format(PORTAL_ID), payload=None, qos=0, retain=False)
        signal.alarm(KEEPALIVE) # call us again in KEEPALIVE seconds
    signal.signal(signal.SIGALRM, keepalive_handler)
    
    client = mqtt.Client()
    client.on_connect = on_connect
    client.on_message = on_message
    client.connect(CCGX_ADDR, 1883, 60)
    keepalive_handler(None, None) # Trigger keepalive
    client.loop_forever()
