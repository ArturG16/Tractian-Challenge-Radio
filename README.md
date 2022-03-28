| Supported Targets | ESP32 |
| ----------------- | ----- |

Tractian Challenge	
========================
I developed a full wireless comunication system for to R&D Challenge from Tractian.
The main objective of this project is send a 500 kB file from a 100 m open-air gap, using at least one battery powered device.
So to reach that goal I choose to use the Bluetooth LE protocol because it has a low power consumption and great range, also because it is suitable to the new versions of it and different hardwares plataforms.

The hardware that I choose was the ESP32-Pico-D4,which has a solid environment,numerous features with low cost and  which makes this the perfect tool for many of IoT projects and experiments.

The pcb I made it simple but efficient with the SoC with 4Mb flash built-in and a battery case of CR123A,also I inserted a crystal of 32kHz in esp32 to support the Bluetooth Modem low power clock which make a system has an average current about 2-3mA. That model of battery has an estimated ~1500mAh and a small size.I used the MAX40200 wich is an ideal diode which has lower dropout voltage <40mV to keep the power supply safe.I didn't put in the project any leds or other peripherals because the focus was to reach the main goal to send messages and be realiabe in battery.

![3D_PCB.png](https://github.com/ArturG16/Tractian-Challenge-Radio/blob/main/PCB/3D_PCB.png)

The firmware I used an example from the manufacturer framework ( ESP - IDF ) which gave me the base to develop a efficient code.
That example used was the GATT Server, which is ideal to use on sensor's projects. Also that framework has the capability to use FreeRToS which brings to the project a great realiable and provides methods for multiple threads or tasks, mutexes, semaphores and software timers. A tickless mode is provided for low power applications.And also it has greater modularity and less module interdependencies facilitates code reuse across projects. 

Therefore the firmware logic is written below:

In the main code was setup the callbacks,initialization of flash and set the Tx Bluetooth radio at maximum power.To reach the goal of 100meters range.
```c
    esp_ble_tx_power_set(ESP_BLE_PWR_TYPE_DEFAULT,ESP_PWR_LVL_P9); 
    esp_ble_tx_power_set(ESP_BLE_PWR_TYPE_ADV, ESP_PWR_LVL_P9); 
    esp_ble_tx_power_set(ESP_BLE_PWR_TYPE_SCAN ,ESP_PWR_LVL_P9); 
```
Then I created a while(1) loop with an 20ms inter task delay just to not trigger the watchdog timer.

After that I created a task when the smartphone( or any other device) after pairing with the esp32 ble node, send a notification message to receive the 500kb data from it..

Later on in the ESP_GATTS_WRITE_EVT case event, that part of code is located in inside callback when the node receives a message,so I commented the original routine to send a only message in response and created the task "ble_notify_task"

```c
xTaskCreate(ble_notify_task, "ble_notify_task", 16384,  &task_arg, 16, NULL); 
```
Inside the task the logics consists in read 500bytes of data, which can be a data storaged in flash or read from somewhere else and after that start to send.
```c
for (int i = 0; i < sizeof(notify_data); ++i) notify_data[i] = i; 
ESP_LOGI(GATTS_TAG, "Sending....%d/1024",cnt); // For Debug
    //the size of notify_data[] need less than MTU size
esp_ble_gatts_send_indicate(ble_addr[0], ble_addr[1], ble_addr[2],sizeof(notify_data), notify_data, false);
```
The time spent to send all the packets was 50seconds.

But the goal of project is send 500kb of data , then I created a counter of each packet of 500bytes was send it and put a limit to 1024 which corresponds a kbyte. After the counter reachs the target number , the loop is break and the task was created is deleted.

```c
    if(cnt>1023)	notifyDone = true;
    if(notifyDone==true) break; // Total time spent 51.150 ms.
	}
	vTaskDelete(NULL);
```

Another thing that helped me to simulated if the code was running correctly was to use my Bluetooth Sniffer using the nRF52840 dongle and the ios App from Nordic ( nRF connected ). Which is the reason to insert a 50ms delay inter packets,because if there isn't that the reception will be compromised.

```c
    vTaskDelay(50/portTICK_RATE_MS); // Task delay necessary to send data correctly.
```

In that project I didnt used the sleep function because it can be suggestive accordingly with the project, because we can have a numerous ways to wake-up the system.
