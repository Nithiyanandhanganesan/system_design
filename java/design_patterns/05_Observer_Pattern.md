# Observer Pattern - Complete Guide

## Introduction

The **Observer Pattern** is a behavioral design pattern that defines a one-to-many dependency between objects. When one object (Subject) changes state, all dependent objects (Observers) are automatically notified and updated. It's also known as the "Publish-Subscribe" pattern.

## Definition

> **Observer Pattern**: Defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

## Problem Solved

- **Loose Coupling**: Reduce coupling between subjects and observers
- **Dynamic Relationships**: Add/remove observers at runtime
- **Event Handling**: Implement event-driven architectures
- **Model-View Separation**: Separate data model from presentation
- **Real-time Updates**: Notify multiple components of state changes
- **Broadcast Communication**: One-to-many communication pattern

## Structure

```
┌─────────────────┐    ┌─────────────────┐
│    Subject      │───▶│    Observer     │
├─────────────────┤    ├─────────────────┤
│ +attach()       │    │ +update()       │
│ +detach()       │    └─────────────────┘
│ +notify()       │              △
└─────────────────┘              │
          △                      │
          │                      │
┌─────────────────┐    ┌─────────────────┐
│ ConcreteSubject │    │ ConcreteObserver│
├─────────────────┤    ├─────────────────┤
│ -state          │    │ +update()       │
│ +getState()     │    └─────────────────┘
│ +setState()     │
└─────────────────┘
```

## Basic Implementation

### Subject Interface and Implementation

```java
/**
 * Observer interface
 */
public interface Observer {
    void update(Subject subject);
}

/**
 * Subject interface
 */
public interface Subject {
    void attach(Observer observer);
    void detach(Observer observer);
    void notifyObservers();
}

/**
 * Concrete Subject - Weather Station
 */
public class WeatherStation implements Subject {
    
    private final List<Observer> observers = new ArrayList<>();
    private double temperature;
    private double humidity;
    private double pressure;
    private String location;
    
    public WeatherStation(String location) {
        this.location = location;
    }
    
    @Override
    public void attach(Observer observer) {
        if (!observers.contains(observer)) {
            observers.add(observer);
            System.out.println("Observer attached to " + location + " weather station");
        }
    }
    
    @Override
    public void detach(Observer observer) {
        if (observers.remove(observer)) {
            System.out.println("Observer detached from " + location + " weather station");
        }
    }
    
    @Override
    public void notifyObservers() {
        System.out.println("Notifying " + observers.size() + " observers of weather changes");
        
        for (Observer observer : observers) {
            try {
                observer.update(this);
            } catch (Exception e) {
                System.err.println("Error notifying observer: " + e.getMessage());
            }
        }
    }
    
    public void setWeatherData(double temperature, double humidity, double pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        
        System.out.println(String.format(
            "Weather updated for %s: T=%.1f°C, H=%.1f%%, P=%.1fmbar",
            location, temperature, humidity, pressure
        ));
        
        notifyObservers();
    }
    
    // Getters
    public double getTemperature() { return temperature; }
    public double getHumidity() { return humidity; }
    public double getPressure() { return pressure; }
    public String getLocation() { return location; }
    
    public int getObserverCount() {
        return observers.size();
    }
}
```

### Concrete Observers

```java
/**
 * Current Conditions Display Observer
 */
public class CurrentConditionsDisplay implements Observer {
    
    private final String displayName;
    private double temperature;
    private double humidity;
    
    public CurrentConditionsDisplay(String displayName) {
        this.displayName = displayName;
    }
    
    @Override
    public void update(Subject subject) {
        if (subject instanceof WeatherStation) {
            WeatherStation station = (WeatherStation) subject;
            this.temperature = station.getTemperature();
            this.humidity = station.getHumidity();
            display();
        }
    }
    
    public void display() {
        System.out.println(String.format(
            "[%s] Current conditions: %.1f°C and %.1f%% humidity",
            displayName, temperature, humidity
        ));
    }
    
    public double getTemperature() { return temperature; }
    public double getHumidity() { return humidity; }
}

/**
 * Statistics Display Observer
 */
public class StatisticsDisplay implements Observer {
    
    private final String displayName;
    private final List<Double> temperatures = new ArrayList<>();
    private final List<Double> humidityReadings = new ArrayList<>();
    private final List<Double> pressureReadings = new ArrayList<>();
    
    public StatisticsDisplay(String displayName) {
        this.displayName = displayName;
    }
    
    @Override
    public void update(Subject subject) {
        if (subject instanceof WeatherStation) {
            WeatherStation station = (WeatherStation) subject;
            
            temperatures.add(station.getTemperature());
            humidityReadings.add(station.getHumidity());
            pressureReadings.add(station.getPressure());
            
            display();
        }
    }
    
    public void display() {
        if (temperatures.isEmpty()) {
            System.out.println("[" + displayName + "] No data available yet");
            return;
        }
        
        System.out.println(String.format(
            "[%s] Statistics - Avg Temp: %.1f°C, Min: %.1f°C, Max: %.1f°C (over %d readings)",
            displayName,
            temperatures.stream().mapToDouble(Double::doubleValue).average().orElse(0.0),
            temperatures.stream().mapToDouble(Double::doubleValue).min().orElse(0.0),
            temperatures.stream().mapToDouble(Double::doubleValue).max().orElse(0.0),
            temperatures.size()
        ));
    }
    
    public double getAverageTemperature() {
        return temperatures.stream().mapToDouble(Double::doubleValue).average().orElse(0.0);
    }
    
    public int getReadingCount() {
        return temperatures.size();
    }
}

/**
 * Forecast Display Observer
 */
public class ForecastDisplay implements Observer {
    
    private final String displayName;
    private double currentPressure = 29.92;
    private double lastPressure = 29.92;
    
    public ForecastDisplay(String displayName) {
        this.displayName = displayName;
    }
    
    @Override
    public void update(Subject subject) {
        if (subject instanceof WeatherStation) {
            WeatherStation station = (WeatherStation) subject;
            
            lastPressure = currentPressure;
            currentPressure = station.getPressure();
            
            display();
        }
    }
    
    public void display() {
        String forecast;
        
        if (currentPressure > lastPressure) {
            forecast = "Improving weather on the way!";
        } else if (currentPressure == lastPressure) {
            forecast = "More of the same";
        } else {
            forecast = "Watch out for cooler, rainy weather";
        }
        
        System.out.println(String.format(
            "[%s] Forecast: %s (Pressure: %.1f, was: %.1f)",
            displayName, forecast, currentPressure, lastPressure
        ));
    }
    
    public String getCurrentForecast() {
        if (currentPressure > lastPressure) {
            return "Improving";
        } else if (currentPressure == lastPressure) {
            return "Stable";
        } else {
            return "Deteriorating";
        }
    }
}
```

## Enhanced Observer with Event Types

### Event-Based Observer Implementation

```java
/**
 * Weather event types
 */
public enum WeatherEventType {
    TEMPERATURE_CHANGE,
    HUMIDITY_CHANGE,
    PRESSURE_CHANGE,
    SEVERE_WEATHER_ALERT,
    ALL_CHANGES
}

/**
 * Weather event data
 */
public class WeatherEvent {
    private final WeatherEventType type;
    private final Object data;
    private final long timestamp;
    private final String source;
    
    public WeatherEvent(WeatherEventType type, Object data, String source) {
        this.type = type;
        this.data = data;
        this.source = source;
        this.timestamp = System.currentTimeMillis();
    }
    
    // Getters
    public WeatherEventType getType() { return type; }
    public Object getData() { return data; }
    public long getTimestamp() { return timestamp; }
    public String getSource() { return source; }
    
    @Override
    public String toString() {
        return String.format("WeatherEvent{type=%s, source=%s, timestamp=%d}", 
            type, source, timestamp);
    }
}

/**
 * Enhanced observer interface with event types
 */
public interface WeatherObserver {
    void onWeatherEvent(WeatherEvent event);
    Set<WeatherEventType> getInterestedEvents();
}

/**
 * Enhanced weather station with event-based notifications
 */
public class EnhancedWeatherStation {
    
    private final Map<WeatherEventType, List<WeatherObserver>> observersByEvent = new HashMap<>();
    private final String location;
    
    private double temperature;
    private double humidity;
    private double pressure;
    private double lastTemperature;
    private double lastHumidity;
    private double lastPressure;
    
    public EnhancedWeatherStation(String location) {
        this.location = location;
        initializeEventTypes();
    }
    
    private void initializeEventTypes() {
        for (WeatherEventType eventType : WeatherEventType.values()) {
            observersByEvent.put(eventType, new ArrayList<>());
        }
    }
    
    public void subscribe(WeatherObserver observer, WeatherEventType... eventTypes) {
        if (eventTypes.length == 0) {
            eventTypes = new WeatherEventType[]{WeatherEventType.ALL_CHANGES};
        }
        
        for (WeatherEventType eventType : eventTypes) {
            List<WeatherObserver> observers = observersByEvent.get(eventType);
            if (!observers.contains(observer)) {
                observers.add(observer);
                System.out.println(String.format(
                    "Observer subscribed to %s events from %s", eventType, location
                ));
            }
        }
    }
    
    public void unsubscribe(WeatherObserver observer, WeatherEventType... eventTypes) {
        if (eventTypes.length == 0) {
            // Remove from all event types
            for (List<WeatherObserver> observers : observersByEvent.values()) {
                observers.remove(observer);
            }
            System.out.println("Observer unsubscribed from all events from " + location);
        } else {
            for (WeatherEventType eventType : eventTypes) {
                List<WeatherObserver> observers = observersByEvent.get(eventType);
                if (observers.remove(observer)) {
                    System.out.println(String.format(
                        "Observer unsubscribed from %s events from %s", eventType, location
                    ));
                }
            }
        }
    }
    
    private void notifyObservers(WeatherEventType eventType, Object data) {
        WeatherEvent event = new WeatherEvent(eventType, data, location);
        
        // Notify specific event subscribers
        List<WeatherObserver> specificObservers = observersByEvent.get(eventType);
        notifyObserverList(specificObservers, event);
        
        // Notify ALL_CHANGES subscribers
        if (eventType != WeatherEventType.ALL_CHANGES) {
            List<WeatherObserver> allObservers = observersByEvent.get(WeatherEventType.ALL_CHANGES);
            notifyObserverList(allObservers, event);
        }
    }
    
    private void notifyObserverList(List<WeatherObserver> observers, WeatherEvent event) {
        for (WeatherObserver observer : observers) {
            try {
                observer.onWeatherEvent(event);
            } catch (Exception e) {
                System.err.println("Error notifying observer: " + e.getMessage());
            }
        }
    }
    
    public void updateTemperature(double newTemperature) {
        if (this.temperature != newTemperature) {
            this.lastTemperature = this.temperature;
            this.temperature = newTemperature;
            
            TemperatureChangeData changeData = new TemperatureChangeData(
                lastTemperature, newTemperature
            );
            
            notifyObservers(WeatherEventType.TEMPERATURE_CHANGE, changeData);
            
            // Check for severe weather
            checkSevereWeather();
        }
    }
    
    public void updateHumidity(double newHumidity) {
        if (this.humidity != newHumidity) {
            this.lastHumidity = this.humidity;
            this.humidity = newHumidity;
            
            HumidityChangeData changeData = new HumidityChangeData(
                lastHumidity, newHumidity
            );
            
            notifyObservers(WeatherEventType.HUMIDITY_CHANGE, changeData);
        }
    }
    
    public void updatePressure(double newPressure) {
        if (this.pressure != newPressure) {
            this.lastPressure = this.pressure;
            this.pressure = newPressure;
            
            PressureChangeData changeData = new PressureChangeData(
                lastPressure, newPressure
            );
            
            notifyObservers(WeatherEventType.PRESSURE_CHANGE, changeData);
            
            checkSevereWeather();
        }
    }
    
    public void updateAllWeatherData(double temperature, double humidity, double pressure) {
        updateTemperature(temperature);
        updateHumidity(humidity);
        updatePressure(pressure);
    }
    
    private void checkSevereWeather() {
        List<String> alerts = new ArrayList<>();
        
        if (temperature > 40.0) {
            alerts.add("Extreme heat warning");
        } else if (temperature < -20.0) {
            alerts.add("Extreme cold warning");
        }
        
        if (pressure < 28.0) {
            alerts.add("Severe storm approaching");
        }
        
        if (!alerts.isEmpty()) {
            SevereWeatherAlert alert = new SevereWeatherAlert(alerts, temperature, pressure);
            notifyObservers(WeatherEventType.SEVERE_WEATHER_ALERT, alert);
        }
    }
    
    // Getters
    public double getTemperature() { return temperature; }
    public double getHumidity() { return humidity; }
    public double getPressure() { return pressure; }
    public String getLocation() { return location; }
    
    public Map<WeatherEventType, Integer> getObserverCounts() {
        return observersByEvent.entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                entry -> entry.getValue().size()
            ));
    }
    
    // Event data classes
    public static class TemperatureChangeData {
        public final double oldValue;
        public final double newValue;
        
        public TemperatureChangeData(double oldValue, double newValue) {
            this.oldValue = oldValue;
            this.newValue = newValue;
        }
    }
    
    public static class HumidityChangeData {
        public final double oldValue;
        public final double newValue;
        
        public HumidityChangeData(double oldValue, double newValue) {
            this.oldValue = oldValue;
            this.newValue = newValue;
        }
    }
    
    public static class PressureChangeData {
        public final double oldValue;
        public final double newValue;
        
        public PressureChangeData(double oldValue, double newValue) {
            this.oldValue = oldValue;
            this.newValue = newValue;
        }
    }
    
    public static class SevereWeatherAlert {
        public final List<String> alerts;
        public final double currentTemperature;
        public final double currentPressure;
        
        public SevereWeatherAlert(List<String> alerts, double temperature, double pressure) {
            this.alerts = new ArrayList<>(alerts);
            this.currentTemperature = temperature;
            this.currentPressure = pressure;
        }
    }
}
```

### Enhanced Observer Implementations

```java
/**
 * Temperature Alert Observer
 */
public class TemperatureAlertObserver implements WeatherObserver {
    
    private final String name;
    private final double minThreshold;
    private final double maxThreshold;
    
    public TemperatureAlertObserver(String name, double minThreshold, double maxThreshold) {
        this.name = name;
        this.minThreshold = minThreshold;
        this.maxThreshold = maxThreshold;
    }
    
    @Override
    public void onWeatherEvent(WeatherEvent event) {
        if (event.getType() == WeatherEventType.TEMPERATURE_CHANGE) {
            EnhancedWeatherStation.TemperatureChangeData data = 
                (EnhancedWeatherStation.TemperatureChangeData) event.getData();
            
            if (data.newValue < minThreshold || data.newValue > maxThreshold) {
                System.out.println(String.format(
                    "[%s ALERT] Temperature %.1f°C is outside safe range (%.1f - %.1f)!",
                    name, data.newValue, minThreshold, maxThreshold
                ));
                
                sendAlert(data.newValue);
            }
        }
    }
    
    @Override
    public Set<WeatherEventType> getInterestedEvents() {
        return Set.of(WeatherEventType.TEMPERATURE_CHANGE);
    }
    
    private void sendAlert(double temperature) {
        // Simulate sending alert (email, SMS, etc.)
        System.out.println("Sending temperature alert notification...");
    }
}

/**
 * Weather Logger Observer
 */
public class WeatherLogger implements WeatherObserver {
    
    private final String loggerName;
    private final List<String> logs = Collections.synchronizedList(new ArrayList<>());
    
    public WeatherLogger(String loggerName) {
        this.loggerName = loggerName;
    }
    
    @Override
    public void onWeatherEvent(WeatherEvent event) {
        String logEntry = String.format(
            "%s [%s] %s: %s",
            new Date(event.getTimestamp()),
            event.getSource(),
            event.getType(),
            formatEventData(event)
        );
        
        logs.add(logEntry);
        System.out.println("[" + loggerName + " LOG] " + logEntry);
    }
    
    @Override
    public Set<WeatherEventType> getInterestedEvents() {
        return Set.of(WeatherEventType.ALL_CHANGES);
    }
    
    private String formatEventData(WeatherEvent event) {
        Object data = event.getData();
        
        if (data instanceof EnhancedWeatherStation.TemperatureChangeData) {
            EnhancedWeatherStation.TemperatureChangeData tempData = 
                (EnhancedWeatherStation.TemperatureChangeData) data;
            return String.format("%.1f°C -> %.1f°C", tempData.oldValue, tempData.newValue);
        } else if (data instanceof EnhancedWeatherStation.SevereWeatherAlert) {
            EnhancedWeatherStation.SevereWeatherAlert alert = 
                (EnhancedWeatherStation.SevereWeatherAlert) data;
            return "SEVERE: " + String.join(", ", alert.alerts);
        }
        
        return data != null ? data.toString() : "No data";
    }
    
    public List<String> getLogs() {
        return new ArrayList<>(logs);
    }
    
    public void clearLogs() {
        logs.clear();
    }
}

/**
 * Mobile App Observer
 */
public class MobileAppObserver implements WeatherObserver {
    
    private final String appName;
    private final Set<WeatherEventType> subscribedEvents;
    private int notificationCount = 0;
    
    public MobileAppObserver(String appName, WeatherEventType... eventTypes) {
        this.appName = appName;
        this.subscribedEvents = EnumSet.copyOf(Arrays.asList(eventTypes));
    }
    
    @Override
    public void onWeatherEvent(WeatherEvent event) {
        notificationCount++;
        
        switch (event.getType()) {
            case TEMPERATURE_CHANGE:
                handleTemperatureChange(event);
                break;
            case SEVERE_WEATHER_ALERT:
                handleSevereWeatherAlert(event);
                break;
            default:
                handleGenericEvent(event);
                break;
        }
    }
    
    @Override
    public Set<WeatherEventType> getInterestedEvents() {
        return new HashSet<>(subscribedEvents);
    }
    
    private void handleTemperatureChange(WeatherEvent event) {
        EnhancedWeatherStation.TemperatureChangeData data = 
            (EnhancedWeatherStation.TemperatureChangeData) event.getData();
        
        System.out.println(String.format(
            "[%s] Push notification: Temperature changed to %.1f°C in %s",
            appName, data.newValue, event.getSource()
        ));
        
        updateUI(data.newValue);
    }
    
    private void handleSevereWeatherAlert(WeatherEvent event) {
        EnhancedWeatherStation.SevereWeatherAlert alert = 
            (EnhancedWeatherStation.SevereWeatherAlert) event.getData();
        
        System.out.println(String.format(
            "[%s] URGENT NOTIFICATION: %s in %s",
            appName, String.join(", ", alert.alerts), event.getSource()
        ));
        
        showEmergencyNotification(alert.alerts);
    }
    
    private void handleGenericEvent(WeatherEvent event) {
        System.out.println(String.format(
            "[%s] Weather update: %s in %s",
            appName, event.getType(), event.getSource()
        ));
    }
    
    private void updateUI(double temperature) {
        // Simulate UI update
        System.out.println("Updating mobile app UI with new temperature");
    }
    
    private void showEmergencyNotification(List<String> alerts) {
        // Simulate emergency notification
        System.out.println("Showing emergency alert popup in mobile app");
    }
    
    public int getNotificationCount() {
        return notificationCount;
    }
}
```

## Real-World Example: Stock Price Monitoring

### Stock Market Observer Implementation

```java
/**
 * Stock interface
 */
public interface Stock {
    void addObserver(StockObserver observer);
    void removeObserver(StockObserver observer);
    void notifyObservers();
    String getSymbol();
    double getPrice();
}

/**
 * Stock Observer interface
 */
public interface StockObserver {
    void onPriceUpdate(Stock stock, double previousPrice, double newPrice);
    void onVolumeUpdate(Stock stock, long newVolume);
}

/**
 * Concrete Stock implementation
 */
public class StockImpl implements Stock {
    
    private final String symbol;
    private final String companyName;
    private final List<StockObserver> observers = new ArrayList<>();
    
    private double price;
    private double previousPrice;
    private long volume;
    private long previousVolume;
    private double dayHigh;
    private double dayLow;
    
    public StockImpl(String symbol, String companyName, double initialPrice) {
        this.symbol = symbol;
        this.companyName = companyName;
        this.price = initialPrice;
        this.previousPrice = initialPrice;
        this.dayHigh = initialPrice;
        this.dayLow = initialPrice;
    }
    
    @Override
    public void addObserver(StockObserver observer) {
        if (!observers.contains(observer)) {
            observers.add(observer);
            System.out.println("Observer added to " + symbol + " stock");
        }
    }
    
    @Override
    public void removeObserver(StockObserver observer) {
        if (observers.remove(observer)) {
            System.out.println("Observer removed from " + symbol + " stock");
        }
    }
    
    @Override
    public void notifyObservers() {
        for (StockObserver observer : observers) {
            try {
                observer.onPriceUpdate(this, previousPrice, price);
                observer.onVolumeUpdate(this, volume);
            } catch (Exception e) {
                System.err.println("Error notifying stock observer: " + e.getMessage());
            }
        }
    }
    
    public void updatePrice(double newPrice) {
        if (this.price != newPrice) {
            this.previousPrice = this.price;
            this.price = newPrice;
            
            // Update day high/low
            if (newPrice > dayHigh) {
                dayHigh = newPrice;
            }
            if (newPrice < dayLow) {
                dayLow = newPrice;
            }
            
            System.out.println(String.format(
                "%s price updated: $%.2f (was $%.2f)",
                symbol, price, previousPrice
            ));
            
            notifyObservers();
        }
    }
    
    public void updateVolume(long newVolume) {
        this.previousVolume = this.volume;
        this.volume = newVolume;
        
        // Notify only volume observers
        for (StockObserver observer : observers) {
            try {
                observer.onVolumeUpdate(this, newVolume);
            } catch (Exception e) {
                System.err.println("Error notifying volume observer: " + e.getMessage());
            }
        }
    }
    
    // Getters
    @Override
    public String getSymbol() { return symbol; }
    @Override
    public double getPrice() { return price; }
    public String getCompanyName() { return companyName; }
    public long getVolume() { return volume; }
    public double getDayHigh() { return dayHigh; }
    public double getDayLow() { return dayLow; }
    
    public double getPriceChange() {
        return price - previousPrice;
    }
    
    public double getPriceChangePercent() {
        return previousPrice != 0 ? ((price - previousPrice) / previousPrice) * 100 : 0;
    }
}

/**
 * Portfolio Manager Observer
 */
public class PortfolioManager implements StockObserver {
    
    private final String managerName;
    private final Map<String, Integer> holdings = new HashMap<>();
    private final Map<String, Double> averageCosts = new HashMap<>();
    private double totalPortfolioValue = 0.0;
    
    public PortfolioManager(String managerName) {
        this.managerName = managerName;
    }
    
    @Override
    public void onPriceUpdate(Stock stock, double previousPrice, double newPrice) {
        String symbol = stock.getSymbol();
        
        if (holdings.containsKey(symbol)) {
            int shares = holdings.get(symbol);
            double oldValue = shares * previousPrice;
            double newValue = shares * newPrice;
            double change = newValue - oldValue;
            
            totalPortfolioValue += change;
            
            System.out.println(String.format(
                "[%s Portfolio] %s: %d shares, value changed from $%.2f to $%.2f (%.2f%%)",
                managerName, symbol, shares, oldValue, newValue,
                ((newValue - oldValue) / oldValue) * 100
            ));
            
            // Check for significant changes
            if (Math.abs((newPrice - previousPrice) / previousPrice) > 0.05) {
                System.out.println(String.format(
                    "[%s ALERT] Significant price change in %s: %.2f%%",
                    managerName, symbol, ((newPrice - previousPrice) / previousPrice) * 100
                ));
            }
        }
    }
    
    @Override
    public void onVolumeUpdate(Stock stock, long newVolume) {
        // Portfolio managers might monitor volume for liquidity
        if (newVolume > 1000000) { // High volume threshold
            System.out.println(String.format(
                "[%s INFO] High volume detected in %s: %,d shares",
                managerName, stock.getSymbol(), newVolume
            ));
        }
    }
    
    public void addHolding(String symbol, int shares, double averageCost) {
        holdings.put(symbol, holdings.getOrDefault(symbol, 0) + shares);
        averageCosts.put(symbol, averageCost);
        System.out.println(String.format(
            "Added %d shares of %s to %s's portfolio at average cost $%.2f",
            shares, symbol, managerName, averageCost
        ));
    }
    
    public void printPortfolioSummary() {
        System.out.println("\n=== " + managerName + " Portfolio Summary ===");
        System.out.println("Total Portfolio Value: $" + String.format("%.2f", totalPortfolioValue));
        
        holdings.forEach((symbol, shares) -> {
            double avgCost = averageCosts.getOrDefault(symbol, 0.0);
            System.out.println(String.format(
                "  %s: %d shares @ avg $%.2f", symbol, shares, avgCost
            ));
        });
    }
}

/**
 * Trading Bot Observer
 */
public class TradingBot implements StockObserver {
    
    private final String botName;
    private final double buyThreshold;
    private final double sellThreshold;
    private final Map<String, Integer> positions = new HashMap<>();
    
    public TradingBot(String botName, double buyThreshold, double sellThreshold) {
        this.botName = botName;
        this.buyThreshold = buyThreshold;
        this.sellThreshold = sellThreshold;
    }
    
    @Override
    public void onPriceUpdate(Stock stock, double previousPrice, double newPrice) {
        String symbol = stock.getSymbol();
        double changePercent = ((newPrice - previousPrice) / previousPrice) * 100;
        
        if (changePercent <= buyThreshold) {
            executeBuyOrder(symbol, newPrice);
        } else if (changePercent >= sellThreshold && positions.containsKey(symbol)) {
            executeSellOrder(symbol, newPrice);
        }
    }
    
    @Override
    public void onVolumeUpdate(Stock stock, long newVolume) {
        // Trading bots might use volume in their algorithms
        if (newVolume > 2000000) {
            System.out.println(String.format(
                "[%s BOT] High volume signal for %s: %,d shares - monitoring closely",
                botName, stock.getSymbol(), newVolume
            ));
        }
    }
    
    private void executeBuyOrder(String symbol, double price) {
        int sharesToBuy = 100; // Fixed order size for demo
        positions.put(symbol, positions.getOrDefault(symbol, 0) + sharesToBuy);
        
        System.out.println(String.format(
            "[%s BOT] BUY ORDER: %d shares of %s at $%.2f (triggered by %.2f%% drop)",
            botName, sharesToBuy, symbol, price, buyThreshold
        ));
    }
    
    private void executeSellOrder(String symbol, double price) {
        int currentPosition = positions.getOrDefault(symbol, 0);
        if (currentPosition > 0) {
            int sharesToSell = Math.min(100, currentPosition);
            positions.put(symbol, currentPosition - sharesToSell);
            
            System.out.println(String.format(
                "[%s BOT] SELL ORDER: %d shares of %s at $%.2f (triggered by %.2f%% rise)",
                botName, sharesToSell, symbol, price, sellThreshold
            ));
        }
    }
    
    public Map<String, Integer> getPositions() {
        return new HashMap<>(positions);
    }
}

/**
 * Price Alert Observer
 */
public class PriceAlertObserver implements StockObserver {
    
    private final String alertName;
    private final Map<String, Double> priceTargets = new HashMap<>();
    private final Map<String, Double> stopLosses = new HashMap<>();
    
    public PriceAlertObserver(String alertName) {
        this.alertName = alertName;
    }
    
    @Override
    public void onPriceUpdate(Stock stock, double previousPrice, double newPrice) {
        String symbol = stock.getSymbol();
        
        // Check price targets
        Double target = priceTargets.get(symbol);
        if (target != null && newPrice >= target) {
            System.out.println(String.format(
                "[%s ALERT] Price target reached for %s: $%.2f (target was $%.2f)",
                alertName, symbol, newPrice, target
            ));
            priceTargets.remove(symbol); // Remove triggered alert
        }
        
        // Check stop losses
        Double stopLoss = stopLosses.get(symbol);
        if (stopLoss != null && newPrice <= stopLoss) {
            System.out.println(String.format(
                "[%s ALERT] Stop loss triggered for %s: $%.2f (stop was $%.2f)",
                alertName, symbol, newPrice, stopLoss
            ));
            stopLosses.remove(symbol); // Remove triggered alert
        }
    }
    
    @Override
    public void onVolumeUpdate(Stock stock, long newVolume) {
        // Price alerts typically don't care about volume
    }
    
    public void setPriceTarget(String symbol, double target) {
        priceTargets.put(symbol, target);
        System.out.println(String.format(
            "Price target set for %s: $%.2f", symbol, target
        ));
    }
    
    public void setStopLoss(String symbol, double stopLoss) {
        stopLosses.put(symbol, stopLoss);
        System.out.println(String.format(
            "Stop loss set for %s: $%.2f", symbol, stopLoss
        ));
    }
}
```

## Observer with Java Built-in Support

### Using Java's Observable and Observer (Deprecated but Educational)

```java
import java.util.Observable;
import java.util.Observer;

/**
 * Using Java's built-in Observable (deprecated since Java 9)
 * Shown for educational purposes
 */
@Deprecated
public class NewsAgency extends Observable {
    
    private String news;
    private String category;
    
    public void setNewsAndNotify(String news, String category) {
        this.news = news;
        this.category = category;
        
        setChanged(); // Mark as changed
        notifyObservers(new NewsUpdate(news, category)); // Notify with data
    }
    
    public String getNews() { return news; }
    public String getCategory() { return category; }
    
    public static class NewsUpdate {
        public final String news;
        public final String category;
        public final long timestamp;
        
        public NewsUpdate(String news, String category) {
            this.news = news;
            this.category = category;
            this.timestamp = System.currentTimeMillis();
        }
    }
}

@Deprecated
public class NewsChannel implements Observer {
    
    private final String channelName;
    
    public NewsChannel(String channelName) {
        this.channelName = channelName;
    }
    
    @Override
    public void update(Observable o, Object arg) {
        if (o instanceof NewsAgency && arg instanceof NewsAgency.NewsUpdate) {
            NewsAgency.NewsUpdate newsUpdate = (NewsAgency.NewsUpdate) arg;
            
            System.out.println(String.format(
                "[%s] Breaking News in %s: %s",
                channelName, newsUpdate.category, newsUpdate.news
            ));
        }
    }
}
```

### Modern Java Observer with CompletableFuture

```java
/**
 * Modern asynchronous observer implementation
 */
public class AsyncEventPublisher {
    
    private final List<Function<Object, CompletableFuture<Void>>> asyncObservers = new ArrayList<>();
    private final ExecutorService executorService = Executors.newCachedThreadPool();
    
    public void subscribe(Function<Object, CompletableFuture<Void>> observer) {
        asyncObservers.add(observer);
    }
    
    public void unsubscribe(Function<Object, CompletableFuture<Void>> observer) {
        asyncObservers.remove(observer);
    }
    
    public CompletableFuture<Void> publishEvent(Object event) {
        List<CompletableFuture<Void>> futures = asyncObservers.stream()
            .map(observer -> observer.apply(event))
            .collect(Collectors.toList());
        
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
    }
    
    public void shutdown() {
        executorService.shutdown();
    }
}

/**
 * Example usage of async observer
 */
public class AsyncObserverExample {
    
    public void demonstrateAsyncObserver() {
        AsyncEventPublisher publisher = new AsyncEventPublisher();
        
        // Subscribe async observers
        publisher.subscribe(event -> CompletableFuture.runAsync(() -> {
            System.out.println("Async Observer 1 processing: " + event);
            simulateWork(1000);
        }));
        
        publisher.subscribe(event -> CompletableFuture.runAsync(() -> {
            System.out.println("Async Observer 2 processing: " + event);
            simulateWork(500);
        }));
        
        // Publish event and wait for all observers to complete
        CompletableFuture<Void> completion = publisher.publishEvent("Test Event");
        
        completion.thenRun(() -> {
            System.out.println("All observers have completed processing");
        });
    }
    
    private void simulateWork(int milliseconds) {
        try {
            Thread.sleep(milliseconds);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## Spring Boot Integration

### Spring Event-Driven Architecture

```java
/**
 * Spring Application Event
 */
public class WeatherUpdateEvent extends ApplicationEvent {
    
    private final double temperature;
    private final double humidity;
    private final String location;
    
    public WeatherUpdateEvent(Object source, double temperature, double humidity, String location) {
        super(source);
        this.temperature = temperature;
        this.humidity = humidity;
        this.location = location;
    }
    
    // Getters
    public double getTemperature() { return temperature; }
    public double getHumidity() { return humidity; }
    public String getLocation() { return location; }
}

/**
 * Spring Event Publisher Service
 */
@Service
public class WeatherService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void updateWeather(String location, double temperature, double humidity) {
        // Update weather data
        System.out.println(String.format(
            "Weather updated for %s: T=%.1f°C, H=%.1f%%", 
            location, temperature, humidity
        ));
        
        // Publish event
        WeatherUpdateEvent event = new WeatherUpdateEvent(this, temperature, humidity, location);
        eventPublisher.publishEvent(event);
    }
}

/**
 * Spring Event Listeners (Observers)
 */
@Component
public class WeatherEventListeners {
    
    @EventListener
    public void handleWeatherUpdate(WeatherUpdateEvent event) {
        System.out.println(String.format(
            "Weather event received: %s - %.1f°C",
            event.getLocation(), event.getTemperature()
        ));
    }
    
    @EventListener
    @Conditional(TemperatureAlertCondition.class)
    public void handleTemperatureAlert(WeatherUpdateEvent event) {
        if (event.getTemperature() > 35.0 || event.getTemperature() < 0.0) {
            System.out.println(String.format(
                "TEMPERATURE ALERT for %s: %.1f°C",
                event.getLocation(), event.getTemperature()
            ));
        }
    }
    
    @EventListener
    @Async
    public void handleAsyncWeatherProcessing(WeatherUpdateEvent event) {
        // Simulate async processing
        try {
            Thread.sleep(2000);
            System.out.println("Async weather processing completed for " + event.getLocation());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

/**
 * Custom condition for conditional event listener
 */
public class TemperatureAlertCondition implements Condition {
    
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();
        return env.getProperty("weather.alerts.enabled", Boolean.class, true);
    }
}

/**
 * Configuration for async event processing
 */
@Configuration
@EnableAsync
public class EventConfig {
    
    @Bean
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("weather-event-");
        executor.initialize();
        return executor;
    }
}
```

## Testing Observer Pattern

### Unit Tests

```java
/**
 * Tests for Observer Pattern implementations
 */
@ExtendWith(MockitoExtension.class)
class ObserverPatternTest {
    
    @Test
    void testBasicWeatherStationObserver() {
        WeatherStation station = new WeatherStation("Test Location");
        CurrentConditionsDisplay display = new CurrentConditionsDisplay("Test Display");
        StatisticsDisplay stats = new StatisticsDisplay("Test Stats");
        
        // Attach observers
        station.attach(display);
        station.attach(stats);
        
        assertThat(station.getObserverCount()).isEqualTo(2);
        
        // Update weather data
        station.setWeatherData(25.0, 60.0, 30.5);
        
        // Verify observers were updated
        assertThat(display.getTemperature()).isEqualTo(25.0);
        assertThat(display.getHumidity()).isEqualTo(60.0);
        assertThat(stats.getReadingCount()).isEqualTo(1);
        
        // Detach observer
        station.detach(display);
        assertThat(station.getObserverCount()).isEqualTo(1);
    }
    
    @Test
    void testEnhancedWeatherStationWithEvents() {
        EnhancedWeatherStation station = new EnhancedWeatherStation("Enhanced Test");
        TemperatureAlertObserver alertObserver = new TemperatureAlertObserver("Test Alert", 10.0, 30.0);
        WeatherLogger logger = new WeatherLogger("Test Logger");
        
        // Subscribe to specific events
        station.subscribe(alertObserver, WeatherEventType.TEMPERATURE_CHANGE);
        station.subscribe(logger, WeatherEventType.ALL_CHANGES);
        
        // Update temperature
        station.updateTemperature(35.0); // Should trigger alert
        
        assertThat(station.getObserverCounts().get(WeatherEventType.TEMPERATURE_CHANGE)).isEqualTo(1);
        assertThat(station.getObserverCounts().get(WeatherEventType.ALL_CHANGES)).isEqualTo(1);
        assertThat(logger.getLogs()).hasSize(1);
    }
    
    @Test
    void testStockObserver() {
        StockImpl stock = new StockImpl("AAPL", "Apple Inc.", 150.0);
        PortfolioManager portfolio = new PortfolioManager("Test Portfolio");
        TradingBot bot = new TradingBot("Test Bot", -5.0, 5.0);
        
        // Add holdings
        portfolio.addHolding("AAPL", 100, 145.0);
        
        // Add observers
        stock.addObserver(portfolio);
        stock.addObserver(bot);
        
        // Update price
        stock.updatePrice(160.0); // 6.67% increase
        
        // Verify price change calculation
        assertThat(stock.getPrice()).isEqualTo(160.0);
        assertThat(stock.getPriceChangePercent()).isCloseTo(6.67, within(0.1));
        
        // Verify bot positions
        assertThat(bot.getPositions()).containsKey("AAPL");
    }
    
    @Test
    void testObserverErrorHandling() {
        WeatherStation station = new WeatherStation("Error Test");
        
        // Create a mock observer that throws exception
        Observer faultyObserver = mock(Observer.class);
        doThrow(new RuntimeException("Observer error")).when(faultyObserver).update(any());
        
        Observer normalObserver = mock(Observer.class);
        
        station.attach(faultyObserver);
        station.attach(normalObserver);
        
        // Update weather - should not fail due to faulty observer
        assertDoesNotThrow(() -> station.setWeatherData(25.0, 60.0, 30.5));
        
        // Verify normal observer was still called
        verify(normalObserver).update(station);
    }
    
    @Mock
    private ApplicationEventPublisher eventPublisher;
    
    @Test
    void testSpringEventPublishing() {
        WeatherService weatherService = new WeatherService();
        ReflectionTestUtils.setField(weatherService, "eventPublisher", eventPublisher);
        
        weatherService.updateWeather("Test Location", 25.0, 60.0);
        
        // Verify event was published
        verify(eventPublisher).publishEvent(any(WeatherUpdateEvent.class));
    }
}
```

### Performance Tests

```java
/**
 * Performance tests for observer pattern
 */
@SpringBootTest
class ObserverPerformanceTest {
    
    @Test
    void testLargeNumberOfObservers() {
        WeatherStation station = new WeatherStation("Performance Test");
        List<Observer> observers = new ArrayList<>();
        
        // Add 1000 observers
        for (int i = 0; i < 1000; i++) {
            CurrentConditionsDisplay display = new CurrentConditionsDisplay("Display " + i);
            observers.add(display);
            station.attach(display);
        }
        
        assertThat(station.getObserverCount()).isEqualTo(1000);
        
        // Measure notification time
        long startTime = System.nanoTime();
        station.setWeatherData(25.0, 60.0, 30.5);
        long endTime = System.nanoTime();
        
        long notificationTime = (endTime - startTime) / 1_000_000; // Convert to milliseconds
        System.out.println("Notification time for 1000 observers: " + notificationTime + "ms");
        
        // Should complete in reasonable time (less than 100ms)
        assertThat(notificationTime).isLessThan(100);
    }
    
    @Test
    void testConcurrentObserverOperations() throws InterruptedException {
        EnhancedWeatherStation station = new EnhancedWeatherStation("Concurrent Test");
        int numberOfThreads = 10;
        CountDownLatch latch = new CountDownLatch(numberOfThreads);
        
        // Create observers concurrently
        for (int i = 0; i < numberOfThreads; i++) {
            final int threadId = i;
            new Thread(() -> {
                try {
                    WeatherLogger logger = new WeatherLogger("Logger " + threadId);
                    station.subscribe(logger, WeatherEventType.ALL_CHANGES);
                    
                    // Update weather from multiple threads
                    station.updateTemperature(20.0 + threadId);
                } finally {
                    latch.countDown();
                }
            }).start();
        }
        
        latch.await(5, TimeUnit.SECONDS);
        
        // Verify no exceptions and reasonable number of observers
        assertThat(station.getObserverCounts().get(WeatherEventType.ALL_CHANGES)).isEqualTo(numberOfThreads);
    }
}
```

## Best Practices

### ✅ **Do's:**

1. **Use interfaces** for both subjects and observers
2. **Handle exceptions** in observer notifications gracefully
3. **Consider thread safety** in multi-threaded environments
4. **Provide event data** with notifications
5. **Allow observer registration/removal** at runtime
6. **Use weak references** for automatic cleanup when appropriate
7. **Consider async notifications** for long-running observer operations

### ❌ **Don'ts:**

1. **Don't create circular dependencies** between observers and subjects
2. **Don't forget to unregister observers** to prevent memory leaks
3. **Don't make observers dependent on notification order**
4. **Don't perform heavy operations** in observer update methods
5. **Don't ignore error handling** in notification mechanisms
6. **Don't create tight coupling** between specific observers and subjects

## When to Use Observer Pattern

### ✅ **Good Use Cases:**
- **Event-driven systems**: User interfaces, game engines
- **Model-View architectures**: Separating data from presentation
- **Publish-Subscribe systems**: Message brokers, event buses
- **Real-time monitoring**: Stock prices, sensor data, system metrics
- **Notification systems**: Email alerts, push notifications
- **Workflow engines**: Process state changes

### ❌ **Avoid Observer For:**
- **Simple method calls**: Direct method invocation is sufficient
- **Tight coupling scenarios**: When observer and subject are closely related
- **Performance-critical paths**: Observer overhead might be significant
- **Complex state synchronization**: Consider other patterns like State or Mediator

## Related Patterns

- **Mediator**: Centralizes complex communications vs. direct observer notifications
- **Command**: Can be used with Observer for undo/redo functionality
- **Model-View-Controller**: Observer is fundamental to MVC architecture
- **Publisher-Subscriber**: More decoupled variant of Observer pattern

## Summary

The Observer pattern is fundamental for:

**Key Benefits:**
- **Loose Coupling**: Subjects and observers are independent
- **Dynamic Relationships**: Runtime registration/removal of observers
- **Open/Closed Principle**: Easy to add new observers without changing subjects
- **Event-Driven Architecture**: Foundation for reactive systems

**Modern Implementations:**
- **Spring Events**: Built-in event publishing and listening
- **RxJava/Reactive Streams**: Advanced observable patterns
- **CompletableFuture**: Async observer implementations
- **Message Brokers**: Distributed observer patterns

Use Observer when you need to notify multiple objects about state changes in a loosely coupled manner!
