# Алгоритм керування станком пакування спайок (банок фарби)

## 1. Опис обладнання

Станок призначений для:
- Закривання (запресування) кришок на **спайках** (одна спайка = 6 з'єднаних маленьких банок).
- Пакування 4 спайок в один пакет.
- Керування через кнопки START/STOP та індикацію.

### 1.1 Входи (дискретні)

| Назва | Тип | Призначення |
|-------|-----|--------------|
| `BTN_START` | NO | Запуск / відновлення роботи |
| `BTN_STOP`  | NC | Аварійна зупинка / пауза (нормально замкнена) |
| `SENSOR1`   | NO | Датчик на конвеєрі закривання – реагує на **кожну маленьку банку** в спайці |
| `SENSOR2`   | NO | Датчик на пакувальному конвеєрі – фіксує **спайку** в зоні зсуву |
| `SENSOR3`   | NO | Датчик наявності пакетів на складі. **LOW** – пакети закінчилися, **HIGH** – є пакети |

### 1.2 Виходи (керування виконавчими пристроями)

| Назва | Фізичний | Призначення | Тип сигналу |
|-------|----------|-------------|--------------|
| `OUTPUT1` | PV1 | Закривання кришки (запресування) | HIGH на час `CLOSE_CAP_HOLD_TIME_MS` |
| `OUTPUT2` | PV2 | Планка зсуву спайки в бік (накопичення) | HIGH – висунуто, LOW – повернуто |
| `OUTPUT3` | PV3 | Штовхач (заштовхує 4 спайки в пакет) | HIGH на час `PUSHER_EXTEND_TIME_MS` |
| `OUTPUT4` | PV4 | Вертикальне переміщення платформи з присосками | HIGH – вниз, LOW – вгору |
| `OUTPUT5` | PV5 | Горизонтальне переміщення платформи | HIGH – вперед (до зони завантаження), LOW – назад |
| `OUTPUT6` | PV6 | Скидання готового пакета | Імпульс HIGH на `BAG_EJECT_PULSE_MS` |
| `OUTPUT7` | PV7 | Вакуум на присоски | HIGH – вакуум ввімкнено |
| `OUTPUT8` | PV8 | Соленоїдний упор для позиціонування спайок | HIGH – шток піднято, LOW – шток опущено |
| `MOTOR1` | - | Кроковий двигун конвеєра закривання | Імпульси + напрямок |
| `MOTOR2` | - | Кроковий двигун пакувального конвеєра | Ввімкнено/вимкнено (безперервне обертання) |
| `LED1`    | - | Зелена лампа (станок працює) | Постійне – робота, моргання – зупинка/пауза |
| `LED2`    | - | Червона/жовта лампа (немає пакетів) | Світиться, коли `SENSOR3 == LOW` і система на паузі |

## 2. Паралельні задачі

Система має три незалежні задачі, що виконуються в головному циклі (або на перериваннях):

- **Задача A** – керування конвеєром закривання (OUTPUT1, SENSOR1).
- **Задача B** – керування накопиченням 4 спайок та пакуванням (SENSOR2, OUTPUT2, OUTPUT8, OUTPUT3, OUTPUT4-7).
- **Задача C** – моніторинг наявності пакетів (SENSOR3, LED2). Працює на перериваннях або швидкому опитуванні.

Конвеєри **MOTOR1** та **MOTOR2** працюють постійно (крім моментів, коли OUTPUT2 утримує 4 спайки – тоді MOTOR2 можна зупинити, але за описом він не зупиняється).

## 3. Загальна діаграма станів (Mermaid)

```mermaid
stateDiagram-v2
    [*] --> STOPPED: Початковий стан
    
    STOPPED --> RUNNING: Натиснуто START\n(всі home, обнулення)
    RUNNING --> STOPPED: Натиснуто STOP\n(аварійна зупинка)
    RUNNING --> PAUSED: Задача C (немає пакетів)\nпісля завершення пакування
    PAUSED --> RUNNING: Натиснуто START\n(якщо пакети з'явилися)
    
    state RUNNING {
        [*] --> TaskA_Active
        [*] --> TaskB_Active
        [*] --> TaskC_Monitor
        
        state TaskA_Active {
            [*] --> Conveyor1Run
            Conveyor1Run --> WaitSensor1: SENSOR1 = 1 (перша банка в спайці)
            WaitSensor1 --> CapCycle: зупинити MOTOR1, OUTPUT1 ON
            CapCycle --> ResumeConveyor: OUTPUT1 OFF, пауза
            ResumeConveyor --> IgnoreNext5: запустити MOTOR1, ігнорувати 5 імпульсів
            IgnoreNext5 --> Conveyor1Run: після 6-го імпульсу (спайка пройшла)
        }
        
        state TaskB_Active {
            [*] --> SetSolenoid: OUTPUT8 = LOW (для 1-ї спайки)
            SetSolenoid --> WaitSensor2: MOTOR2 рухається
            WaitSensor2 --> ShiftSpayka: SENSOR2 = 1
            ShiftSpayka --> AfterShift: OUTPUT2 HIGH на час SHIFT_TIME
            AfterShift --> IncCounter: spaykasAtPacking++
            IncCounter --> Check4: spaykas == 4?
            Check4 --> HoldOutput2: так (OUTPUT2 залишається HIGH)
            Check4 --> ToggleSolenoid: ні (OUTPUT2 LOW, перемкнути OUTPUT8)
            ToggleSolenoid --> WaitSensor2
            HoldOutput2 --> PrepBag: підготовка пакета (OUTPUT4-7)
            PrepBag --> PushPack: OUTPUT3 HIGH (штовхач)
            PushPack --> RetractBoth: OUTPUT2 LOW, OUTPUT3 LOW
            RetractBoth --> ResetCounters: spaykasAtPacking = 0
            ResetCounters --> SetSolenoid
        }
        
        state TaskC_Monitor {
            [*] --> CheckPackets
            CheckPackets --> NoPackets: SENSOR3 == LOW
            NoPackets --> WaitForPackEnd: якщо packagingBusy, дочекатися
            WaitForPackEnd --> RequestPause: після завершення пакування
            RequestPause --> [*]: перехід до PAUSED
            CheckPackets --> [*]: якщо SENSOR3 == HIGH, нічого не робити
        }
    }
    
    state PAUSED {
        [*] --> WaitStart: LED1 моргає, LED2 світиться (якщо немає пакетів)
        WaitStart --> RUNNING: START + SENSOR3 == HIGH
    }
