# Pozyx-Unity-MQTT
A Unity project connecting to a local Pozyx MQTT positioning stream.


## Setup

#### Prerequisites

To run this project, you will need:

* Pozyx Creator:
	- Pozyx Creator software installed.
	- Running setup with tags being positioned.
* OR Pozyx Enterprise:
	- Pozyx gateway with running tags.
	- The local IP of your gateway
* [Unity 2019.1 or newer](https://unity.com/). I recommend Unity Hub for version management if you prefer to use an older version of Unity. Older versions may work but are untested.

### Installation

Clone the project using `git clone https://github.com/laurentva/Pozyx-Unity-MQTT` and open the project in Unity. Importing the project will take a little while, but after this you should be ready to go.


## Running the project

When opening the project you'll see that the scene named mainScene is already open. This is the scene you need to play.

![](Assets/unity-local-mqtt-getting-started.png)

Press play and you will see different *Tags* that are created in the sidebar. These follow the Prefab you pass to the GameObject object. This way, it's easy to replace the spawned objects and their behavior. Note that their position will be updated whenever you have a new position.

![](Assets/unity-local-mqtt-play.png)

## Inside the code

In this "behind the scenes" bit we'll get more in-depth regarding the code, in a more developer-focused segment.

The C# code we'll need to look at is all located in `Assets/localMqtt.cs`. The code keeps track of what IDs have been seen in the MQTT stream and creates/updates instances of the configured prefab that match a tag's ID. This way you can see multiple tags and create extra behaviour for these objects. 

### Connecting to the MQTT client

For the MQTT functionality in Unity, we use [vovacooper's Unity MQTT library](https://github.com/vovacooper/Unity3d_MQTT). This is a wrapper for Unity using M2Mqtt. This should be included in the project automatically when you clone it.

**Note:**If you have an enterprise setup, or want to run this on another device than the computer running the software, replace the "127.0.0.1" with the local IP of your gateway. 

```cs
// create a MQTT client which will connect to the local MQTT broker.
mqttClient = new MqttClient(IPAddress.Parse("127.0.0.1"), 1883, false, null);

// connect to a callback, this function will be triggered every time a message is received.
mqttClient.MqttMsgPublishReceived += onMqttMessageReceived;

// create a random ID for the client to connect to the broker with.
var clientId = Guid.NewGuid().ToString();
mqttClient.Connect(clientId);

// subscribe to the "tags" topic. This is an MQTT topic that will provide data with the lowest latency.
mqttClient.Subscribe(new[] {"tags"}, new[] {MqttMsgBase.QOS_LEVEL_EXACTLY_ONCE});
```

### Parsing the MQTT messages

We'll use the [JSON .NET Unity asset](https://assetstore.unity.com/packages/tools/input-management/json-net-for-unity-11347) to parse the incoming tag data.

The tag data comes in as an array of individual tag packets. These are documented [here](http://api-docs.cloud.pozyxlabs.com/#/pages/mqtt/tags). 

When a tag has a successful position, we get the data by accessing the JSON format and creating a PositionData object, definition found in `Assets/PositionData.cs.
Since the MQTT callback function can not modify the scene directly, we push these successful positions on a queue which will be emptied and used during Unity's Update() function.


```cs
    private void onMqttMessageReceived(object sender, MqttMsgPublishEventArgs e)
    {
    		// we take the message from the MQTT event and parse it as UTF8.
        var message = Encoding.UTF8.GetString(e.Message);

		// the data comes in as an array.
        var messageData = JArray.Parse(message);
		
		// we turn the parsed message into a proper array.
        var messageObj = JArray.Parse(messageData.ToString());
        
        // iterate over each tag packet
        foreach (var tagData in messageObj)
        {
            // this catches unsuccessful positioning packets
            if (!(bool) tagData["success"]) continue;
			
			// if the packet is successful, we add it to the positionQueue
            var positionData = new PositionData((string) tagData["tagId"], (int) tagData["data"]["coordinates"]["x"],
                (int) tagData["data"]["coordinates"]["y"], (int) tagData["data"]["coordinates"]["z"]);

            positionQueue.Add(positionData);
        }
    }
```


### Update function

In the update function, we add or update the existing objects that are bound to the tag ID. We do this by emptying the positionQueue. Note that we invert y and z in this function call, as we want the positions to resemble the reality, and the y-as is the vertical axis in the Unity system.

```cs

        // The list positionQueue is used as a FIFO queue, is filled by the MQTT callback 
        // and emptied here.
        while (positionQueue.Count > 0)
        {
            var positionData = positionQueue[0];
            Debug.Log("Found with ID " + positionData.ID);
            
            // Modified because of Unity axis system
            AddPosition(positionData.ID, positionData.x, positionData.z, positionData.y);

            // This would use the original Unity system
            // AddPosition(positionData.ID, positionData.x, positionData.y, positionData.z);
            
            // the position is now removed from the queue
            positionQueue.Remove(positionData);
        }
        
```



