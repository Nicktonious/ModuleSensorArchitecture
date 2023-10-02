# ModuleSensorArchitecture
Модуль предоставляет стек классов для работы с сенсорами. Все создаваемые в рамках фреймворка EcoLite модули сенсоров должны быть производными от данного стека.

Набор классов, обеспечивающих функционал датчика можно условно разделить на такие части: 
- Основная, которая состоит из ветки в виде двух обобщенных классов. От этой ветки и наследуется класс конкретного датчика.
- Сервисная, реализующая математико-логический аппарат для обработки и корректировки поступаемых значений. 
- Прикладная - класс, отвечающий за отдельно взятый канал датчика. Этот класс реализуется вне данного стека.

### **ClassAncestorSensor** 
Самый "старший" предок в иерархии классов датчиков. В первую очередь собирает в себе самые базовые данные о датчике: переданные шину, пины и тд. Так же сохраняет его описательную характеристику: имя, тип вх. и вых. сигналов, типы шин которые можно использовать, количество каналов и тд.
Его инициализация происходит либо передачей в конструктор двух объектов: объекта типа **SensorOptsType** и **SensorPropsType** (см.аргументы *_opts* и *_sensor_props* в примерах ниже), либо передачей только первого объекта, с возможностью передать объект **SensorPropsType** непосредственно в метод `InitSensProps(_sensor_props)` в любой другой момент. 

#### **Примеры**
Параметр *_opts* типа **SensorOptsType**:
```
const _opts = {
    bus: some_i2c_bus,      //объект шины
    pins: [B14, B15],       //массив используемых пинов 
    quantityChannel: 2      //количество измерительных каналов
}
```
Пример параметра *_sensor_props* типа **SensorPropsType**: 
```
const _sensor_props = {
    name: "VL6180",                
    type: "sensor",
    channelNames: ['light', 'range'],
    typeInSignal: "analog",
    typeOutSignal: "digital",
    busType: ["i2c"],
    manufacturingData: {
        IDManufacturing: [
            { "Amperka": "AMP-B072" }
        ],
        IDsupplier: [
            { "Amperka": "AMP-B072" }
        ],
        HelpSens: "Proximity sensor"
    }
};
```

Так как ClassAncestorSensor прмиеняется исколючительно в роли родительского класаа, его наследники обязаны иметь такие же параметры конструктора, которые передаются в супер-конструктор таким образом:

`ClassAncestorSensor.apply(this, [_opts, _sensor_props || null]);`

### **ClassMiddleSensor** 
Класс, наследующийся от **ClassAncestorSensor**. Закладывает в будущие классы датчиков поля и методы, необходимые для унификации хранения значений, полученных с каналов. Вводит реализации возможности выделения из объекта "реального" датчика объектов-каналов, которые сам же при инициализации создает и хранит в поле *_Channels*.
Если при инициализации класса-предка тот получил корректное значение *_QuantityChannel*, то:
1. в конструкторе вызываются необходимые инструкции для автоматического создания аксессоров по паттерну `Ch0_Value`, `Ch1_Value` и тд., в зависимости от количества каналов. Данные аксессоры во первых, служат единой прослойкой, через которую проходят данные и соответственно лишь в сеттере можно лакончино и своевременно прогнать данные через лимиты, корректирующие функции и зоны измерения и их обработчики. "Под капотом" геттеров лежит не просто числовая переменная, в которую присваивается значение, а кольцевой буффер, который работает с массивом значений, который в дальнейшем будет передаваться в функции-фильтры. Таким образом в более высокоуровневом коде запись и чтение значений должны выполняться исключительно через эти аксессоры.
Изменение глубины фильтра (макс.кол-ва значений, хранящихся в буффере совершается вызовом метода `SetFilterDepth(_num)`)
2. поле-массив *_Channels* заполняется объектами типа **ClassChannel**. 

Еще одной важнейшей целью класса является определение списика сигнатур основных методов, часть из которых так же перебрасывается в класс-канал датчика. Список возможных методов:
- `Init(_opts)` 
    Метод обязывающий провести инициализацию датчика настройкой необходимых для его работы регистров 
- `Start(_ch_num, _period, _opts)`
    Метод обязывает запустить циклический опрос определенного канала датчика с заданной периодичностью в мс. Переданное значение периода должно сверяться с минимальным значением для данного канала и, если требуется, регулироваться, так как максимальная частота опроса зависит от характеристик датичка. 

    Для передачи дополнительных аргументов, может передаваться объект _opts
- `Stop(_ch_num)`
    Метод обязывает прекратить считывание значений с заданного канала. 
    В случаях, когда значения данного канала считываются синхронно с другими, достаточно прекратить обновление данных.
- `ChangeFreq(_ch_num, _period)`
    Метод обязывает останавливить опрос указанного канала и запустить его вновь с уже новой частотой. Возобновиться должно обновление всех каналов, которые опрашивались перед остановкой.
- `Run(_ch_num, _opts)`
    Метод который обязывает запустить прикладную работу датчика, сперва выполнив его полную инициализацию, конфигурацию и прочие необходимые процедуры, обеспечив его безопасный и корректный запуск
- `ConfigureRegs(_opts)`
    Метод обязывающий выполнить дополнительную конфигурацию датчика. Это может быть настройка пина прерывания, периодов измерения и прочих шагов, которые в общем случае необходимы для работы датчика, но могут переопределяться в процессе работы, и потому вынесены из метода Init() 
- `Reset()`
    Метод обязывающий выполнить перезагрузку датчика
- `SetRepeatability(_rep)`
    Метод устанавливающий значение повторяемости
- `SetPrecision(_pres)`
    Метод устанавливающий точность измерений
- `Read(_reg)`
    Метод обеспечивающий чтение с регистра
- `Write(_reg, _val)`
    Метод обеспечивающий запись в регистр

Подробнее о каждом методе в соответствующих комментариях к *ClassSensorArchitecture.js*

### **ClassChannel** 
Класс, представляющий каждый отдельно взятый канал датчика. При чем, каждый канал является "синглтоном" для своего родителя. Хранит в себе ссылки на основной объект сенсора (поле *_ThisSensor*) и "проброшенные" методы для работы с данным каналом датчика, включая аксессоры. Таким образом, в перую очередь класс представляет собой интерфейс для работы с датчиком. 

А так же в полях *_DataRefine* и *_Alarms* этого классе инстанцируются сервисные классы **ClassDataRefine** и **ClassDataRefine**, которые безусловно используются в аксессорах класса **ClassMiddleSensor** при обработке считываемых с датчика значений.

Предусматривается что именно через объект этого класса пользователь запускает и контроллирует прикладную работу датчика. 
Список "проброшенных" с **ClassMiddleSensor** методов:
- `Start(_period, _opts)`
- `Stop()`
- `ChangeFreq(_period)`
- `Run(_opts)`
- `ConfigureRegs(_opts)`
- `Reset()`
- `SetFilterDepth(_num)`

#### **ClassDataRefine** 
Класс реализующий функционал для обработки выходных числовых значений:
1. супрессию значения по задаваемым ограничителям (лимитам)
2. корректировку соответственно заданной линейной функции kx+b
3. хранение фильтр-функции

Установка ограничителей происходит вызовом метода `SetOutLim(_limLow, _limHigh)`, в котором *_limLow* и *_limHigh* - 2 упорядоченных в порядке возрастания числа.

Установка коэффициентов линейной функции происходит через метод `SetTransmissonOut(_k, _b)`. 

Фильтр-функция устанавливается методом `SetFilterFunc(_func)`.

Методы `SupressOutValue(val)` и `CalibrateOutValue(val)` используются аксессором *ChN_Value* **ClassMiddleSensor**-а.

### **ClassAlarms** 
Класс реализующий функционал для работы зонами измерения. Хранит в себе заданные границы зон измерения и алармы (коллбэки, вызываемые при переходе величины выходного значения датчика из одной зоны в другую). 
Границы желтой и красной зон определяются вручную, а диапазон зеленой зоны фактически подстраивается под желтую (или красную если желтая не определена).

Установка желтой и красной измерительных зон и их функций-обработчиков происходит вызовом мотода `SetYellowZone(_low, _high, _callbackLow, _callbackHigh)`, в котором *_low* и *_high* - 2 упорядоченных в порядке возрастания числа (нижняя и верхняя границы), а *_callbackLow* и *_callbackHigh* - коллбэки, вызываемые при получении датчиком значения, попадающего в данную зону "сверху" и соответсвенно "снизу". 


Конфигурация класса выполняется методом `SetZones(_opts)`, в котором _opts - объект со свойствами *red*, *yellow*, *green*, которые в виде объекта определяют параметры конкретной зоны. 
Объекты *red* и *yellow* имеют такие свойства:
- low - числовое значение границы нижней зоны, всегда ниже границы верхней зоны 
- high - границы верхней зоны
- cbLow - коллбэк (аларм) нижней зоны
- cbHigh - аларм верхней зоны

Так как границы зеленой зоны зависят от границ желтой либо красной зон, то объект *green* передает только единичный аларм для зеленой зоны в свойстве *cb*.

Правила, нюансы задания значений:
- red.low < yellow.low
- red.high > yellow.high
- при повторном вызове `SetZones()` проверка значений на валидность происходит таким образом: 
    1. новые значения желтой/красной зон сверяются со значениями красной/желтой зон если такие также были переданы
    2. если же была передана только красная либо желтая зона, то ее значения сверяются со значениями зон, указанными прежде. 


#### **Примеры**:
задание всех зон за раз:
```
someSensor1.SetZones({         
    red: { low: -100, high: 70, cbLow: () => { console.log('L_RED'); }, cbHigh: () => { console.log('H_RED'); }},
    yellow: { low: -50, high: 50, cbLow: () => { console.log('L_YELLOW'); }, cbHigh: () => { console.log('H_YELLOW')} },
    green: { cb: () => { console.log('GREEN'); } }
});
```
также можно опускать задание некоторых зон:
```
someSensor2.SetZones({          
    red: { low: 0, high: 170, cbLow: () => { console.log('L_RED'); }, cbHigh: () => { console.log('H_RED'); }},
    green: { cb: () => { console.log('GREEN'); } }
});

someSensor3.SetZones({         
    yellow: { low: -50, high: 50, cbLow: () => { console.log('L_YELLOW'); }, cbHigh: () => { console.log('H_YELLOW')} },
    green: { cb: () => { console.log('GREEN'); } }
});
```

