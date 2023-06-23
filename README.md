# LivoloReadRF433 for Home Assistant
Чтение кодов **RF433** брелоков от фирмы **Livolo** с помощью **ESPHome**.
Reading codes **RF433** key fobs from **Livolo** using **ESPHome**.

## О проекте

**Задача:** Использование пульта Livolo работающего на частоте 433МГц,
 в автоматизациях умного дома Home Assistant. Для работы используется ESPHome.

**Используемые компоненты:**
- ESP8266-01S 

![Фото](photo\ESP8266-01S.webp)

- Приемник на 433МГц QIACHIP RX500A

<img src="photo\QIACHIP RX500A top1.webp" width="320"/> 

- Питание на Ваш вкус. Я использую Hi-Link HLP-PM01 (5В, 0.6А)

<img src="photo\hi-link-pm01-3w.jpg" width="320"/> 
<!--// height="300" />-->

- Пульт Livolo

<img src="photo\photo_1.jpg" width="320"/> 

<!--<img src="photo\photo_2.jpg" width="320"/> -->
<img src="photo\photo_3.jpg" width="320"/> 

<img src="photo\photo_4.jpg" width="320"/> 



 
## Подключение:

Один из контактов приемника **(DO)** подключается к контакту **GPIO2** ESP8266-01S.

<img src="photo\QIACHIP RX500A 433MGz top.jpg" width="301"/> 
<img src="photo\ESP8266-01S-pinout.jpg" width="320"/> 


## Прошивка:

- В ESPHome создаем новое устройство, называем (например) **livoloreadrf433**.
Прошиваем его стартовой прошивкой.
Затем кладем файлик LivoloReadRF433.h по пути **config\esphome\\**
рядом с **livoloreadrf433.yaml**

- Редактируем файл настроек **livoloreadrf433.yaml**
  - прописываем в начале файла:

    ```yaml
    esphome:
      includes:
        - LivoloReadRF433.h

    ```

  - и добавляем сенсоры:

    ```yaml
    sensor:
    - platform: custom
      lambda: |-
        auto my_sensor = new LivoloReadRF433();
        App.register_component(my_sensor);
        return {my_sensor->RemoteID ,my_sensor->KeyCode};

      sensors:
        - name: "RemoteID"
          force_update: true
        - name: "KeyCode"    
          force_update: true
    ```

    В результате в интеграции ESPHome появится устройство LivoloReadRF433.

## Пример автоматизации:

```yaml
alias: >-
  Большая комната. Свет. Переключение одной лампочки по кнопке "А" пульта Livolo 
description: >-
  Переключаем одну лампочку в большой комнате при нажатии на кнопку "А" пультика Livolo
trigger:
  - platform: state
    entity_id:
      - sensor.livoloreadrf433_keycode  #такой сенсор будет создан, если имя устройства задали livoloreadrf433
    from: null
    to: "8" #код кнопки
condition:
  - condition: state
    entity_id: sensor.livoloreadrf433_remoteid #такой сенсор будет создан, если имя устройства задали livoloreadrf433
    state: "32375" #код пульта
action:
  - type: toggle
    device_id: 38ec008bb918814e2981071e5c2b6022  #здесь прописываете ваш выключатель
    entity_id: switch.switch_3
    domain: switch
    enabled: true
  - service: python_script.set_state #данное действие пришлось добавить, из-за некорректного отрабатывания параметра force_update: true
    data_template:
      entity_id: sensor.livoloreadrf433_keycode
      state: "100" #принудительно устанавливаем записываем в сенсор keycode "100".
    enabled: true
mode: single
```
## Дополнение:
При тестировании обнаружилась проблема: при нажатии на одну и туже кнопку пульта, в Home Assistant передавалось только первое нажатие. Если нажимать разные кнопки - всё передавалось нормально.

Т.е. в логе ESPHome устройства был виден перехват кода кнопки, но т.к. его значение не изменялось, передача в HA не осуществлялась.

Для исправления данной ситуации был использован параметр "force_update: true", который должен был заставить ESPHome передавать данные, даже если код клавиши не изменился.

Но, выяснилось, что параметр не отрабатывает как задумывался (по крайней мере у меня). [Почитать об этом можно сдесь.](https://github.com/home-assistant/core/issues/91221)

В результате пришлось добавить принудительное изменение значения 
сенсора **keycode** (записываем, значение, которое заведомо не совпадает с любым кодом клавиш).

Для включения возможности изменения значений датчиков через автоматизации, надо:
1. В configuration.yaml добавить "python_script:"
2. В папке config (в ней лежит configuration.yaml) создать папку "python_scripts"
3. В папку "python_scripts" положить файл "set_state.py"
4. Перезагрузить HA.

Более подробно описано в статье (статья не моя):
https://kvvhost.ru/2020/05/03/python-set-state-ha/

Код обработчика нажатий клавиш пульта взят здесь:
https://forum.arduino.cc/t/control-livolo-switches-livolo-switch-library/149850
