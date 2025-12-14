# Sterownik Roweru Elektrycznego - Model AADL

> Projekt z przedmiotu Systemy Czasu Rzeczywistego  
> Autor: Piotr Kaniuk  
> Email: baniuk@student.agh.edu.pl

## Spis treści
- [Opis systemu](#opis-systemu)
- [Opis funkcjonalny](#opis-funkcjonalny)
- [Architektura](#architektura)
- [Specyfikacja komponentów](#specyfikacja-komponentów)
- [Diagramy systemu](#diagramy-systemu)
- [Analiza przepływów](#analiza-przepływów)
- [Wyniki analiz](#wyniki-analiz)
- [Założenia techniczne](#założenia-techniczne)
- [Możliwe rozszerzenia](#możliwe-rozszerzenia)
- [Literatura](#literatura)

---

## Opis systemu

Projekt przedstawia kompleksowy model AADL (Architecture Analysis & Design Language) systemu sterowania roweru elektrycznego. Model obejmuje pełną architekturę elektronicznego układu sterowania, w tym: wejścia sensoryczne, jednostki przetwarzające, sterowanie silnikiem oraz infrastrukturę komunikacyjną.

System został zbudowany w architekturze rozproszonej Master-Slave:
- **Master (Controller Subsystem)**: Główna jednostka sterująca odpowiedzialna za akwizycję danych z czujników, interfejs użytkownika i podejmowanie decyzji wysokiego poziomu
- **Slave (Motor Subsystem)**: Dedykowana jednostka sterowania silnikiem obsługująca operacje napędowe w czasie rzeczywistym

### Kluczowe cechy
- Harmonogramowanie zadań okresowych w czasie rzeczywistym
- Komunikacja magistralą CAN pomiędzy podsystemami
- Zarządzanie energią z systemem BMS (Battery Management System)
- Wiele wejść sensorycznych (prędkość, kadencja, moment obrotowy)
- Interfejs użytkownika z wyświetlaczem LCD i przyciskami sterującymi
- Funkcje bezpieczeństwa (czujniki hamulców, system alarmowy)
- Łączność (GPS, Bluetooth)

---

## Opis funkcjonalny

### Perspektywa użytkownika

System sterowania roweru elektrycznego zapewnia płynną integrację między działaniem rowerzysty a wspomaganiem silnika:

1. **Akwizycja danych z czujników**
   - Czujnik prędkości monitoruje obroty koła
   - Czujnik kadencji śledzi szybkość pedałowania
   - Czujnik momentu obrotowego mierzy siłę pedałowania
   - Czujniki Halla dostarczają informacji o pozycji wirnika silnika
   - Czujniki hamulców wykrywają hamowanie

2. **Zarządzanie energią**
   - System zarządzania baterią (BMS) monitoruje poziom naładowania i napięcie
   - Dynamiczna alokacja mocy w zależności od stanu baterii
   - Możliwość odzysku energii przy hamowaniu (regenerative braking)

3. **Przetwarzanie sterowania**
   - Główny sterownik przetwarza wszystkie dane z czujników co 100ms
   - Wątek sterowania silnikiem działa co 50ms dla szybkiej reakcji
   - Algorytmy obliczają optymalny poziom wspomagania silnika
   - Sprawdzenia bezpieczeństwa zapewniają wyłączenie silnika przy hamowaniu

4. **Interfejs użytkownika**
   - Wyświetlacz LCD pokazuje prędkość, dystans, poziom baterii i kadencję
   - Przyciski sterujące umożliwiają wybór trybu jazdy
   - Oświetlenie LED dla widoczności
   - Brzęczyk alarmowy dla ostrzeżeń

5. **Sterowanie silnikiem**
   - Silnik BLDC sterowany sygnałami PWM
   - Kontroler mocy zarządza prądem silnika do 1000W
   - Płynne dostarczanie momentu obrotowego w oparciu o dane wejściowe

### Przepływ operacji systemu

```
Czujniki → Główny sterownik → Komenda silnika → Podsystem silnika → Silnik BLDC
   ↓              ↓                                        ↓
BMS baterii   Wyświetlacz/UI                   Sprzężenie Hall/Prąd
```

---

## Architektura

### Hierarchia systemu

Model AADL jest zorganizowany w trzech głównych warstwach:

#### 1. **Warstwa urządzeń (Device Layer)**
Komponenty sprzętowe łączące się ze światem fizycznym:
- Czujniki (Speed, Cadence, Torque, Hall, Current)
- Aktuatory (Silnik BLDC, Oświetlenie LED, Alarm)
- Interfejs użytkownika (Wyświetlacz LCD, Przyciski sterujące)
- Moduły komunikacyjne (GPS, Bluetooth)
- Komponenty zasilania (BMS baterii, Kontroler mocy)

#### 2. **Warstwa procesów/wątków (Process/Thread Layer)**
Komponenty programowe realizujące logikę sterowania:
- **Main_Control_Thread**: Główny algorytm sterowania (Okres: 100ms)
- **Motor_Control_Thread**: Sterowanie silnikiem w czasie rzeczywistym (Okres: 50ms)

#### 3. **Warstwa platformy (Platform Layer)**
Infrastruktura obliczeniowa i komunikacyjna:
- **Main_CPU**: Procesor 200 MIPS dla podsystemu sterownika
- **Motor_CPU**: Procesor 80 MIPS dla podsystemu silnika
- **Pamięć**: Flash (1MB) i RAM (256KB) dla głównego sterownika
- **Magistrale**: CAN (1 MBps), SPI (10 MBps), System Bus (100 MBps)

### Architektura komunikacji


```
┌─────────────────────────┐      Magistrala CAN      ┌──────────────────┐
│  Controller Subsystem   │◄────────────────────────►│ Motor Subsystem  │
│                         │                          │                  │
│  - Main CPU (200 MIPS)  │   Motor_Command.impl     │  - Motor CPU     │
│  - Czujniki SPI         │─────────────────────────►│  - Silnik BLDC   │
│  - BMS (CAN)            │                          │  - Czujniki Hall │
│  - Wyświetlacz i UI     │                          │  - Sterowanie PWM│
└─────────────────────────┘                          └──────────────────┘
         │
         │ Magistrala zasilania (kable 16mm^2 o ratingu 60A, napięcie baterii - 48V -> 2880W) 
         ▼
   Bateria główna
```

---

## Specyfikacja komponentów

### Typy danych

#### `Ride_Data.impl`
Złożona struktura danych zawierająca informacje o jeździe:
- `Speed`: Aktualna prędkość (Float)
- `Distance`: Przebyta odległość (Float)
- `Cadence`: Kadencja pedałowania (Integer)
- `Battery_Level`: Procent naładowania baterii (Float)

#### `Motor_Command.impl`
Struktura komendy sterującej silnikiem:
- `Motor_Power`: Poziom mocy PWM (Integer)
- `Assist_Mode`: Wybór trybu wspomagania (Integer)
- `Regen_Braking`: Włączenie odzysku energii (Boolean)

### Magistrale komunikacyjne

| Typ magistrali | Przepustowość | Opóźnienie | Przeznaczenie |
|----------------|---------------|------------|---------------|
| **CAN_Bus** | 1.0 MBps | 1-3 ms | Komunikacja między podsystemami |
| **SPI_Bus** | 10.0 MBps | 2-5 ms | Interfejs czujników i wyświetlacza |
| **System_Bus** | 100.0 MBps | 1-5 ns | Wewnętrzna komunikacja procesor-pamięć |
| **Power_Bus** | 2880W | - | Dystrybucja zasilania |

### Urządzenia - Controller Subsystem

| Urządzenie | Interfejs | Moc | Opóźnienie | Opis |
|------------|-----------|-----|------------|------|
| **Speed_Sensor** | SPI | 0.15W | 10-20ms | Pomiar prędkości koła |
| **Cadence_Sensor** | SPI | 0.1W | - | Prędkość obrotu pedałów |
| **Torque_Sensor** | SPI | 0.2W | - | Pomiar siły pedałowania |
| **Battery_BMS** | CAN | 0.5W | - | System zarządzania baterią |
| **LCD_Display** | SPI | 0.8W | - | Wyświetlacz interfejsu użytkownika |
| **Control_Buttons** | Bezpośredni | 0.05W | - | Przyciski wyboru trybu |
| **LED_Light** | Bezpośredni | 3.0W | - | Oświetlenie przednie/tylne |
| **GPS_Module** | SPI | 0.5W | - | Śledzenie lokalizacji |
| **Bluetooth_Module** | SPI | 0.3W | - | Połączenie ze smartfonem |
| **Alarm_Buzzer** | Bezpośredni | 0.2W | - | Ostrzeżenia dźwiękowe |
| **Front_Brake** | Bezpośredni | 0.05W | - | Czujnik hamulca przedniego |
| **Rear_Brake** | Bezpośredni | 0.05W | - | Czujnik hamulca tylnego |

### Urządzenia - Motor Subsystem

| Urządzenie | Interfejs | Moc | Opóźnienie | Opis |
|------------|-----------|-----|------------|------|
| **BLDC_Motor** | PWM | 1000W | 10-30ms | BLDC |
| **Power_Controller** | Bezpośredni | 5.0W | - | Stopień mocy silnika |
| **Hall_Sensor** | Bezpośredni | 0.1W | - | Pozycja wirnika silnika |
| **Current_Sensor** | Bezpośredni | 0.2W | - | Monitorowanie prądu silnika |

### Wątki

#### `Main_Control_Thread`
- **Okres**: 100 ms
- **Czas wykonania**: 8-20 ms
- **Wejścia**: Prędkość, kadencja, moment, bateria, GPS, Bluetooth, przyciski, hamulce
- **Wyjścia**: Komenda silnika, dane LCD, sterowanie LED, alarm
- **Przepływy**: 
  - `motor_path`: Wejście prędkości → Generowanie komendy silnika
  - `lcd_path`: Wejście prędkości → Wyjście wyświetlacza

#### `Motor_Control_Thread`
- **Okres**: 50 ms
- **Czas wykonania**: 3-8 ms
- **Wejścia**: Komenda silnika, RPM, prąd
- **Wyjścia**: Sygnał PWM
- **Przepływ**: `pwm_path`: Komenda → Generowanie PWM

### Procesy

#### `Controller_Process.impl`
- **Budżet MIPS**: 100.0 MIPS
- **Zawiera**: Main_Control_Thread
- **Przepływy**: Przetwarzanie danych z czujników i generowanie komend

#### `Motor_Process.impl`
- **Budżet MIPS**: 30.0 MIPS
- **Zawiera**: Motor_Control_Thread
- **Przepływy**: Sterowanie silnikiem w czasie rzeczywistym

### Procesory

#### `Main_CPU`
- **Wydajność**: 200 MIPS
- **Okres zegara**: 10,000 ps (100 MHz)
- **Moc**: 3.0W
- **Interfejsy**: System Bus, SPI Bus, CAN Bus, Power Bus

#### `Motor_CPU`
- **Wydajność**: 80 MIPS
- **Moc**: 1.0W
- **Interfejsy**: CAN Bus, Local Bus, Power Bus

### Pamięć

#### `Large_Memory.impl` (Główny sterownik)
- **Flash**: 1 MB (pamięć programu)
- **RAM**: 256 KB (dane runtime)

#### `Small_Memory` (Sterownik silnika)
- **Pamięć lokalna** dla firmware sterowania silnikiem

---

## Diagramy systemu

### Diagram instancji - System kompletny

```
E_Bike_System.impl
├── Controller_Subsystem.impl
│   ├── Main_CPU (200 MIPS)
│   ├── Virtual Processor
│   ├── Controller_Process.impl
│   │   └── Main_Control_Thread (okres 100ms)
│   ├── Large_Memory.impl (Flash 1MB + RAM 256KB)
│   ├── System_Bus (100 MBps)
│   ├── SPI_Bus (10 MBps)
│   └── Urządzenia:
│       ├── Speed_Sensor
│       ├── Cadence_Sensor
│       ├── Torque_Sensor
│       ├── Battery_BMS
│       ├── LCD_Display
│       ├── Control_Buttons
│       ├── LED_Light
│       ├── GPS_Module
│       ├── Bluetooth_Module
│       ├── Alarm_Buzzer
│       ├── Front_Brake
│       └── Rear_Brake
│
├── Motor_Subsystem.impl
│   ├── Motor_CPU (80 MIPS)
│   ├── Virtual Processor
│   ├── Motor_Process.impl
│   │   └── Motor_Control_Thread (okres 50ms)
│   ├── Small_Memory
│   ├── Local_Bus
│   └── Urządzenia:
│       ├── BLDC_Motor (1000W)
│       ├── Power_Controller (5W)
│       ├── Hall_Sensor
│       └── Current_Sensor
│
├── CAN_Bus (1 MBps)
├── Power_Bus (2880W)
└── Main_Battery (3kg)
```

### Diagram przepływu danych

![E-Bike System Diagram](E_Bike_System.png)

---

## Analiza przepływów

### Przepływy End-to-End

#### 1. **Przepływ sterowania E2E** (Controller Subsystem)
```
Speed_Sensor.speed_source 
  → połączenie d_spd 
  → Controller_Process.motor_flow 
  → połączenie d_mtr 
  → command_output
```

**Cel**: Główna ścieżka sterowania od pomiaru prędkości do generowania komendy silnika

#### 2. **Globalny przepływ systemu** (System kompletny)
```
Controller_Subsystem.source_flow 
  → command_link (CAN Bus) 
  → Motor_Subsystem.sink_flow
```

**Cel**: Transmisja komend między podsystemami

#### 3. **Przepływ wykonania silnika** (Motor Subsystem)
```
command_input 
  → Motor_Process.proc_path 
  → BLDC_Motor.motor_sink
```

**Cel**: Od komendy silnika do fizycznej aktywacji

---

## Wyniki analiz

### Analiza Flow Latency

**Ustawienia analizy:**
- Typ systemu: System synchroniczny (SS)
- Polityka partycji: Major frame delayed (MF)
- Przypadek pesymistyczny: Maksymalny czas wykonania (ET)
- Przypadek optymistyczny: Pusta kolejka (EQ)
- Opóźnienie kolejkowania: Włączone

**Wyniki dla przepływu 'controller_module.e2e_control' systemu 'E_Bike_System.impl':**

| Element przepływu | Min Spec | Min Act | Metoda Min | Max Spec | Max Act | Metoda Max | Komentarze |
|-------------------|----------|---------|------------|----------|---------|------------|------------|
| **device speed_sensor** | 0.0ms | 0.0ms | no sampling/queuing latency | 0.0ms | 0.0ms | no sampling/queuing latency | |
| **bus spi_bus** | 2.0ms | 2.0ms | specified | 5.0ms | 5.0ms | specified | Użyto określonego opóźnienia magistrali |
| **connection speed_out → speed_in** | 0.0ms | 0.0ms | no sampling/queuing latency | 0.0ms | 0.0ms | no sampling/queuing latency | Dodawanie opóźnień z protokołów i sprzętu |
| **partition controller_module** | 0.0ms | 0.0ms | partition major frame | 0.0ms | 0.0ms | partition major frame | Partycja n Max (S) major frame 0.0ms |
| **thread control_thread (motor_path)** | 0.0ms | 100.0ms | sampling | 0.0ms | 100.0ms | sampling | Okres próbkowania 100.0ms |
| **thread control_thread (motor_path)** | 0.0ms | 8.0ms | processing time | 0.0ms | 20.0ms | processing time | |
| **Latency Total** | 2.0ms | 110.0ms | | 5.0ms | 125.0ms | |

**Określone opóźnienie End-to-End:** 0.0ms

**WARNING**: Expected end to end latency is not specified (Oczekiwane opóźnienie end-to-end nie zostało określone)

**Analiza wyników:**
- Minimalne opóźnienie: 110 ms (2 ms magistrala + 100 ms próbkowanie + 8 ms przetwarzanie)
- Maksymalne opóźnienie: 125 ms (5 ms magistrala + 100 ms próbkowanie + 20 ms przetwarzanie)
- Dominujący czynnik: Okres próbkowania wątku sterowania (100 ms)
- Opóźnienie jest akceptowalne dla systemu roweru elektrycznego (czas reakcji człowieka ~200ms)

### Analiza wymagań mocy (Power Requirements)

**Analiza dla 'power_cable' w systemie E_Bike_System.impl:**

```
Pojemność:  2880.0 W
Zasilanie:  0.0 mW
Budżet:     2030.0 W

Całkowity budżet power_cable: 2030.0 W (mieści się w pojemności 2880.0 W)
```

**Szczegółowy bilans mocy:**

| Podsystem/Urządzenie | Budżet mocy | Procent |
|----------------------|-------------|---------|
| **Controller Subsystem** | ~10W | 0.35% |
| - Main_CPU | 3.0W | |
| - Czujniki (Speed, Cadence, Torque) | 0.45W | |
| - Battery_BMS | 0.5W | |
| - LCD_Display | 0.8W | |
| - LED_Light | 3.0W | |
| - GPS + Bluetooth | 0.8W | |
| - Pozostałe (przyciski, alarm, hamulce) | 0.4W | |
| **Motor Subsystem** | ~1006W | 8.9% |
| - Motor_CPU | 1.0W | |
| - BLDC_Motor | 1000.0W | |
| - Power_Controller | 5.0W | |
| - Czujniki (Hall, Current) | 0.3W | |
| **Całkowity budżet** | **2030W** | **70.5%** |
| **Rezerwa** | 850W | 29.5% |

**Wnioski:**
- System mieści się w limicie pojemności magistrali zasilania (2880W)
- Wykorzystanie: 70.5% pojemności
- Rezerwa bezpieczeństwa: 850W (29.5%)
- Dominujący odbiór: Silnik BLDC (1000W = 49% całkowitego budżetu 2030W)
- Elektronika sterująca: ~13W (bardzo energooszczędna)

**Uwagi dotyczące analizy mocy:**
- Budżet 2030W uwzględnia maksymalną moc wszystkich komponentów jednocześnie
- W rzeczywistości silnik nie zawsze pracuje na pełnej mocy
- Typowe zużycie podczas jazdy: 250-500W (tryb eco) do 1000W (maksymalna moc)
- Rezerwa 850W pozwala na przyszłe rozszerzenia systemu

---

## Założenia techniczne

### Ograniczenia czasu rzeczywistego
1. **Wykonanie okresowe**: Wszystkie wątki używają okresowego protokołu dispatchingu
2. **Rate Monotonic**: Zadania o wyższej częstotliwości (sterowanie silnikiem) mają wyższy priorytet
3. **Komunikacja synchroniczna**: Porty danych zapewniają deterministyczną komunikację
4. **Brak alokacji dynamicznej**: Wszystkie zasoby przydzielane statycznie w fazie projektowania

### Budżet mocy
- **Całkowita moc systemu**: < 2880W w normalnej pracy
- **Szczytowa moc silnika**: 1000W
- **Moc sterownika**: ~3W
- **Czujniki i UI**: ~5W
- **Margines bezpieczeństwa**: 850W dla ładowania baterii/rezerw

### Budżet obliczeniowy
- **Wykorzystanie Main CPU**: 100 MIPS / 200 MIPS = 50%
- **Wykorzystanie Motor CPU**: 30 MIPS / 80 MIPS = 37.5%
- Oba w bezpiecznych granicach z zapasem na szczyty obciążenia

### Komunikacja
- **Obciążenie magistrali CAN**: Komendy silnika (~10 Hz) + dane BMS (1 Hz) << 1 MBps pojemności
- **Obciążenie magistrali SPI**: Wiele czujników dzielących przepustowość 10 MBps
- **Granice opóźnień**: Wszystkie komunikacje kończą się w jednej głównej ramce

### Funkcje bezpieczeństwa
1. **Wyłącznik hamulca**: Każde hamowanie natychmiast zeruje moc silnika
2. **Ochrona baterii**: BMS monitoruje napięcie/prąd aby zapobiec uszkodzeniom
3. **Watchdog**: Procesory wirtualne mogą implementować monitoring watchdog
4. **Fail-Safe**: System domyślnie ustawia zerową moc silnika przy każdej awarii

---

## Możliwe rozszerzenia

### Ulepszenia krótkoterminowe
1. **Analiza możliwości harmonogramowania**
   - Dodanie szczegółowych właściwości czasowych do wszystkich wątków
   - Weryfikacja wykonalności harmonogramowania Rate Monotonic
   - Obliczenie czasów odpowiedzi w najgorszym przypadku

2. **Obsługa błędów**
   - Modelowanie trybów awarii czujników
   - Dodanie redundantnych ścieżek sensorycznych
   - Implementacja wykrywania i izolacji błędów

3. **Zaawansowana analiza przepływów**
   - Kompletne specyfikacje przepływów dla wszystkich ścieżek
   - Dodanie budżetów opóźnień do połączeń
   - Przeprowadzenie kompleksowej analizy opóźnień end-to-end

### Rozszerzenia długoterminowe
1. **Zaawansowane algorytmy sterowania**
   - System wspomagania pedałowania (PAS) z wieloma poziomami
   - Tryb wspomagania podjazdu
   - Funkcjonalność tempomat
   - Optymalizacja odzysku energii przy hamowaniu

2. **Funkcje łączności**
   - Integracja z aplikacją mobilną przez Bluetooth
   - Śledzenie trasy i nawigacja GPS
   - Aktualizacje firmware over-the-air
   - Śledzenie antykradzieżowe

3. **Optymalizacja energetyczna**
   - Dynamiczne zarządzanie mocą w zależności od terenu
   - Predykcyjne obliczanie zasięgu
   - Profile trybów Eco/Sport/Turbo

4. **Ulepszenia bezpieczeństwa**
   - Automatyczne sterowanie światłami w zależności od oświetlenia otoczenia
   - System ostrzegania przed kolizją
   - Protokół zatrzymania awaryjnego
   - Logowanie diagnostyczne

5. **Migracja platformy**
   - Generowanie kodu dla platform wbudowanych
   - Integracja z AUTOSAR dla niezawodności klasy automotive
   - Partycjonowanie ARINC653 dla certyfikacji bezpieczeństwa

---

## Literatura

### Standardy i specyfikacje
- **SAE AS5506C** - Architecture Analysis & Design Language (AADL)
- **ISO 4210** - Wymagania bezpieczeństwa dla rowerów
- **EN 15194** - Rowery ze wspomaganiem elektrycznym (EPAC)

### Narzędzia i dokumentacja
- **OSATE** (Open Source AADL Tool Environment) - Główne narzędzie modelowania
- **OSATE User Guide** - Dokumentacja narzędzia i procedur analizy
- **SEI AADL Resources** - Przewodniki Carnegie Mellon Software Engineering Institute

### Prace powiązane
- Wzorce projektowe systemów wbudowanych czasu rzeczywistego
- Architektury systemów sterowania automotive
- Metody weryfikacji systemów bezpieczeństwa krytycznego

---

## Autor

**Piotr Kaniuk**  
baniuk@student.agh.edu.pl  
Akademia Górniczo-Hutnicza im. Stanisława Staszica w Krakowie

