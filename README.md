# AI-enabled-Sales-Predictor-Web-App-Dashboard
AI enabled Sales Predictor Web App Dashboard using Java and Spring Boot


Nice — I’ll give you a **complete, runnable Java + Spring Boot + MySQL** project that builds an **AI-enabled (simple forecasting) sales prediction dashboard** for an MSME retail shop.
This is a practical, production-friendly starter you can run locally on your Inspiron (VS Code). The predictor uses **lightweight time-series methods** (moving average and exponential smoothing) implemented in Java so you can run everything in one stack. I also show where to plug in advanced ML models (Python service or Smile/ONNX) later.

What you’ll get in this answer:

* Full **project structure**
* All important **code files** (pom.xml, entities, repos, services, controllers, Thymeleaf UI)
* **Forecast algorithms** (moving average & exponential smoothing) in Java
* **Docker Compose** (MySQL + app) and run instructions
* Notes on how to extend to stronger AI models (optional Python ML microservice / callouts)

---

# 1 — Project overview & structure

```
ai-sales-predictor/
├── pom.xml
├── Dockerfile
├── docker-compose.yml
├── src/main/java/com/example/sales/
│   ├── SalesApplication.java
│   ├── config/AppConfig.java
│   ├── controller/WebController.java
│   ├── controller/ApiController.java
│   ├── domain/SalesRecord.java
│   ├── domain/Prediction.java
│   ├── repository/SalesRecordRepository.java
│   ├── repository/PredictionRepository.java
│   ├── service/ForecastService.java
│   └── service/CsvService.java
├── src/main/resources/
│   ├── application.yml
│   └── templates/index.html
└── README.md
```

This is intentionally minimal so you can extend quickly.

---

# 2 — Build file (pom.xml)

```xml
<!-- pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>ai-sales-predictor</artifactId>
  <version>0.0.1</version>
  <properties>
    <java.version>17</java.version>
    <spring.boot.version>3.1.6</spring.boot.version>
  </properties>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring.boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <!-- Spring Boot -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- MySQL -->
    <dependency>
      <groupId>com.mysql</groupId>
      <artifactId>mysql-connector-j</artifactId>
      <scope>runtime</scope>
    </dependency>

    <!-- CSV parsing -->
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-csv</artifactId>
      <version>1.10.0</version>
    </dependency>

    <!-- Jackson -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
    </dependency>

    <!-- Lombok (optional - helps with boilerplate) -->
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

---

# 3 — Configuration (application.yml)

`src/main/resources/application.yml`

```yaml
spring:
  datasource:
    url: jdbc:mysql://db:3306/ai_sales?createDatabaseIfNotExist=true&useSSL=false&allowPublicKeyRetrieval=true
    username: root
    password: password
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        format_sql: true

server:
  port: 8080

app:
  forecast:
    defaultDays: 7
```

Notes: docker-compose maps `db` hostname. If running locally without Docker, change URL to `localhost`.

---

# 4 — Main app & config

`src/main/java/com/example/sales/SalesApplication.java`

```java
package com.example.sales;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SalesApplication {
    public static void main(String[] args) {
        SpringApplication.run(SalesApplication.class, args);
    }
}
```

---

# 5 — Domain entities

`src/main/java/com/example/sales/domain/SalesRecord.java`

```java
package com.example.sales.domain;

import jakarta.persistence.*;
import java.time.LocalDate;

@Entity
@Table(indexes = {
    @Index(columnList = "storeId,itemId,date", name = "idx_store_item_date")
})
public class SalesRecord {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long storeId;
    private Long itemId;
    private LocalDate date;
    private Double sales;   // units sold or revenue
    private Double price;
    private Integer promotion; // 0/1

    // getters & setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public Long getStoreId() { return storeId; }
    public void setStoreId(Long storeId) { this.storeId = storeId; }

    public Long getItemId() { return itemId; }
    public void setItemId(Long itemId) { this.itemId = itemId; }

    public LocalDate getDate() { return date; }
    public void setDate(LocalDate date) { this.date = date; }

    public Double getSales() { return sales; }
    public void setSales(Double sales) { this.sales = sales; }

    public Double getPrice() { return price; }
    public void setPrice(Double price) { this.price = price; }

    public Integer getPromotion() { return promotion; }
    public void setPromotion(Integer promotion) { this.promotion = promotion; }
}
```

`src/main/java/com/example/sales/domain/Prediction.java`

```java
package com.example.sales.domain;

import jakarta.persistence.*;
import java.time.LocalDate;

@Entity
public class Prediction {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long storeId;
    private Long itemId;
    private LocalDate date;
    private Double predictedSales;
    private String model; // "moving_average" or "exp_smoothing"

    // getters & setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public Long getStoreId() { return storeId; }
    public void setStoreId(Long storeId) { this.storeId = storeId; }

    public Long getItemId() { return itemId; }
    public void setItemId(Long itemId) { this.itemId = itemId; }

    public LocalDate getDate() { return date; }
    public void setDate(LocalDate date) { this.date = date; }

    public Double getPredictedSales() { return predictedSales; }
    public void setPredictedSales(Double predictedSales) { this.predictedSales = predictedSales; }

    public String getModel() { return model; }
    public void setModel(String model) { this.model = model; }
}
```

---

# 6 — Repositories

`src/main/java/com/example/sales/repository/SalesRecordRepository.java`

```java
package com.example.sales.repository;

import com.example.sales.domain.SalesRecord;
import org.springframework.data.jpa.repository.JpaRepository;
import java.time.LocalDate;
import java.util.List;

public interface SalesRecordRepository extends JpaRepository<SalesRecord, Long> {
    List<SalesRecord> findByStoreIdAndItemIdOrderByDateAsc(Long storeId, Long itemId);
    List<SalesRecord> findByStoreIdAndItemIdAndDateBetweenOrderByDateAsc(Long storeId, Long itemId, LocalDate start, LocalDate end);
    boolean existsByStoreIdAndItemIdAndDate(Long storeId, Long itemId, LocalDate date);
}
```

`src/main/java/com/example/sales/repository/PredictionRepository.java`

```java
package com.example.sales.repository;

import com.example.sales.domain.Prediction;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface PredictionRepository extends JpaRepository<Prediction, Long> {
    List<Prediction> findByStoreIdAndItemIdOrderByDateAsc(Long storeId, Long itemId);
    void deleteByStoreIdAndItemId(Long storeId, Long itemId);
}
```

---

# 7 — CSV service (ingest CSV)

`src/main/java/com/example/sales/service/CsvService.java`

```java
package com.example.sales.service;

import com.example.sales.domain.SalesRecord;
import com.example.sales.repository.SalesRecordRepository;
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVRecord;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.InputStreamReader;
import java.io.Reader;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;

@Service
public class CsvService {
    private final SalesRecordRepository repo;

    public CsvService(SalesRecordRepository repo) {
        this.repo = repo;
    }

    // Expected CSV header: date,store_id,item_id,sales,price,promotion
    public int ingest(MultipartFile file) throws Exception {
        try (Reader r = new InputStreamReader(file.getInputStream())) {
            CSVParser parser = new CSVParser(r, CSVFormat.DEFAULT.withFirstRecordAsHeader().withTrim());
            List<SalesRecord> toSave = new ArrayList<>();
            for (CSVRecord rec : parser) {
                SalesRecord s = new SalesRecord();
                s.setDate(LocalDate.parse(rec.get("date")));
                s.setStoreId(Long.parseLong(rec.get("store_id")));
                s.setItemId(Long.parseLong(rec.get("item_id")));
                s.setSales(Double.parseDouble(rec.get("sales")));
                s.setPrice(rec.isMapped("price") && !rec.get("price").isBlank() ? Double.parseDouble(rec.get("price")) : 0.0);
                s.setPromotion(rec.isMapped("promotion") && !rec.get("promotion").isBlank() ? Integer.parseInt(rec.get("promotion")) : 0);

                // avoid duplicate insertion for same date/store/item
                if (!repo.existsByStoreIdAndItemIdAndDate(s.getStoreId(), s.getItemId(), s.getDate())) {
                    toSave.add(s);
                }
            }
            repo.saveAll(toSave);
            return toSave.size();
        }
    }
}
```

---

# 8 — Forecast service (prediction algorithms implemented in Java)

`src/main/java/com/example/sales/service/ForecastService.java`

```java
package com.example.sales.service;

import com.example.sales.domain.Prediction;
import com.example.sales.domain.SalesRecord;
import com.example.sales.repository.PredictionRepository;
import com.example.sales.repository.SalesRecordRepository;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Provides simple forecasting methods:
 *  - moving average (window days)
 *  - simple exponential smoothing (alpha)
 *
 * This is intentionally simple to run inside Java. Replace with better ML (Python microservice) later.
 */
@Service
public class ForecastService {
    private final SalesRecordRepository salesRepo;
    private final PredictionRepository predRepo;

    @Value("${app.forecast.defaultDays:7}")
    private int defaultDays;

    public ForecastService(SalesRecordRepository salesRepo, PredictionRepository predRepo) {
        this.salesRepo = salesRepo;
        this.predRepo = predRepo;
    }

    public List<Prediction> forecastMovingAverage(Long storeId, Long itemId, int days, int window) {
        List<SalesRecord> history = salesRepo.findByStoreIdAndItemIdOrderByDateAsc(storeId, itemId);
        if (history.isEmpty()) return Collections.emptyList();

        // Map by date
        Map<LocalDate, Double> map = history.stream()
                .collect(Collectors.toMap(SalesRecord::getDate, SalesRecord::getSales, Double::sum, TreeMap::new));

        // Create ordered list of dates -> values
        List<LocalDate> dates = new ArrayList<>(map.keySet());
        List<Double> values = dates.stream().map(map::get).collect(Collectors.toList());

        List<Prediction> preds = new ArrayList<>();
        for (int i = 0; i < days; i++) {
            // moving avg of last 'window' values
            int start = Math.max(0, values.size() - window);
            List<Double> windowVals = values.subList(start, values.size());
            double avg = windowVals.stream().mapToDouble(Double::doubleValue).average().orElse(0.0);
            LocalDate predDate = dates.get(dates.size() - 1).plusDays(i + 1);
            Prediction p = new Prediction();
            p.setStoreId(storeId);
            p.setItemId(itemId);
            p.setDate(predDate);
            p.setPredictedSales(avg);
            p.setModel("moving_average");
            preds.add(p);

            // append predicted value into values for iterative prediction
            values.add(avg);
            dates.add(predDate);
        }

        // persist: remove previous predictions for this store+item and save new ones
        predRepo.deleteByStoreIdAndItemId(storeId, itemId);
        predRepo.saveAll(preds);
        return preds;
    }

    public List<Prediction> forecastExpSmoothing(Long storeId, Long itemId, int days, double alpha) {
        List<SalesRecord> history = salesRepo.findByStoreIdAndItemIdOrderByDateAsc(storeId, itemId);
        if (history.isEmpty()) return Collections.emptyList();

        List<Double> values = history.stream().map(SalesRecord::getSales).collect(Collectors.toList());
        List<LocalDate> dates = history.stream().map(SalesRecord::getDate).collect(Collectors.toList());

        // initial level: first value
        double level = values.get(0);
        for (int i = 1; i < values.size(); i++) {
            double obs = values.get(i);
            level = alpha * obs + (1 - alpha) * level;
        }

        List<Prediction> preds = new ArrayList<>();
        LocalDate lastDate = dates.get(dates.size() - 1);
        double forecastValue = level;
        for (int i = 1; i <= days; i++) {
            forecastValue = level; // simple SES forecast repeats level
            LocalDate predDate = lastDate.plusDays(i);
            Prediction p = new Prediction();
            p.setStoreId(storeId);
            p.setItemId(itemId);
            p.setDate(predDate);
            p.setPredictedSales(forecastValue);
            p.setModel("exp_smoothing");
            preds.add(p);

            // for iterative approach we could update level with the forecasted value (but SES usually repeats)
        }

        predRepo.deleteByStoreIdAndItemId(storeId, itemId);
        predRepo.saveAll(preds);
        return preds;
    }

    // convenience wrapper using defaultDays
    public List<Prediction> forecastDefaultMovingAvg(Long storeId, Long itemId) {
        return forecastMovingAverage(storeId, itemId, defaultDays, 7);
    }
}
```

Notes:

* This `ForecastService` implements **moving average** and **simple exponential smoothing**. These are lightweight, interpretable, and work well for many small-business use cases.
* Later you can replace `forecastExpSmoothing` with a call to a more advanced Python time-series model (Prophet, ARIMA, XGBoost) via HTTP endpoints.

---

# 9 — Controllers (API + Web)

`src/main/java/com/example/sales/controller/ApiController.java`

```java
package com.example.sales.controller;

import com.example.sales.domain.Prediction;
import com.example.sales.repository.PredictionRepository;
import com.example.sales.service.CsvService;
import com.example.sales.service.ForecastService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.util.List;

@RestController
@RequestMapping("/api")
public class ApiController {
    private final CsvService csvService;
    private final ForecastService forecastService;
    private final PredictionRepository predictionRepository;

    public ApiController(CsvService csvService, ForecastService forecastService, PredictionRepository predictionRepository) {
        this.csvService = csvService;
        this.forecastService = forecastService;
        this.predictionRepository = predictionRepository;
    }

    @PostMapping("/upload")
    public ResponseEntity<?> uploadCsv(@RequestParam("file") MultipartFile file) {
        try {
            int inserted = csvService.ingest(file);
            return ResponseEntity.ok("Inserted " + inserted + " new records");
        } catch (Exception e) {
            e.printStackTrace();
            return ResponseEntity.badRequest().body("Failed: " + e.getMessage());
        }
    }

    @PostMapping("/forecast/mavg")
    public ResponseEntity<?> forecastMovingAvg(@RequestParam Long storeId, @RequestParam Long itemId,
                                               @RequestParam(defaultValue = "7") int days,
                                               @RequestParam(defaultValue = "7") int window) {
        List<Prediction> preds = forecastService.forecastMovingAverage(storeId, itemId, days, window);
        return ResponseEntity.ok(preds);
    }

    @PostMapping("/forecast/exp")
    public ResponseEntity<?> forecastExp(@RequestParam Long storeId, @RequestParam Long itemId,
                                         @RequestParam(defaultValue = "7") int days,
                                         @RequestParam(defaultValue = "0.3") double alpha) {
        List<Prediction> preds = forecastService.forecastExpSmoothing(storeId, itemId, days, alpha);
        return ResponseEntity.ok(preds);
    }

    @GetMapping("/predictions")
    public ResponseEntity<?> getPredictions(@RequestParam Long storeId, @RequestParam Long itemId) {
        return ResponseEntity.ok(predictionRepository.findByStoreIdAndItemIdOrderByDateAsc(storeId, itemId));
    }
}
```

`src/main/java/com/example/sales/controller/WebController.java`

```java
package com.example.sales.controller;

import com.example.sales.domain.Prediction;
import com.example.sales.repository.PredictionRepository;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

import java.util.List;

@Controller
public class WebController {
    private final PredictionRepository predictionRepository;

    public WebController(PredictionRepository predictionRepository) {
        this.predictionRepository = predictionRepository;
    }

    @GetMapping("/")
    public String index() {
        return "index";
    }

    // simple page to fetch predictions via JS; keep server side minimal
}
```

---

# 10 — Frontend (Thymeleaf + Chart.js)

`src/main/resources/templates/index.html`

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <title>MSME Sales Predictor</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.1/dist/css/bootstrap.min.css" rel="stylesheet"/>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body class="bg-light">
<div class="container py-4">
  <h2 class="mb-3">MSME Sales Predictor — Dashboard</h2>

  <div class="card mb-3">
    <div class="card-body">
      <h5>Upload historical sales CSV</h5>
      <form id="uploadForm">
        <div class="mb-3">
          <input type="file" id="fileInput" accept=".csv" class="form-control"/>
        </div>
        <button class="btn btn-primary" type="submit">Upload</button>
        <span id="uploadMsg" class="ms-3"></span>
      </form>
      <small class="text-muted">CSV header: date,store_id,item_id,sales,price,promotion</small>
    </div>
  </div>

  <div class="card mb-3">
    <div class="card-body">
      <h5>Run Forecast</h5>
      <div class="row g-2">
        <div class="col-md-2">
          <input id="storeId" class="form-control" placeholder="storeId" value="1"/>
        </div>
        <div class="col-md-2">
          <input id="itemId" class="form-control" placeholder="itemId" value="1001"/>
        </div>
        <div class="col-md-2">
          <input id="days" class="form-control" placeholder="days" value="14"/>
        </div>
        <div class="col-md-2">
          <select id="modelSel" class="form-select">
            <option value="mavg">Moving Average</option>
            <option value="exp">Exp Smoothing</option>
          </select>
        </div>
        <div class="col-md-4">
          <button id="runBtn" class="btn btn-success">Run Forecast</button>
          <span id="predictMsg" class="ms-3"></span>
        </div>
      </div>
    </div>
  </div>

  <div class="card">
    <div class="card-body">
      <h5>History & Predictions</h5>
      <canvas id="chart" height="120"></canvas>
    </div>
  </div>
</div>

<script>
  const uploadForm = document.getElementById('uploadForm');
  const fileInput = document.getElementById('fileInput');
  const uploadMsg = document.getElementById('uploadMsg');
  const runBtn = document.getElementById('runBtn');
  const predictMsg = document.getElementById('predictMsg');
  let chart;

  uploadForm.addEventListener('submit', async (e) => {
    e.preventDefault();
    const f = fileInput.files[0];
    if (!f) { uploadMsg.innerText = 'Select file'; return; }
    const fd = new FormData();
    fd.append('file', f);
    uploadMsg.innerText = 'Uploading...';
    const res = await fetch('/api/upload', { method: 'POST', body: fd });
    const txt = await res.text();
    uploadMsg.innerText = txt;
  });

  runBtn.addEventListener('click', async () => {
    const storeId = document.getElementById('storeId').value;
    const itemId = document.getElementById('itemId').value;
    const days = document.getElementById('days').value;
    const model = document.getElementById('modelSel').value;

    predictMsg.innerText = 'Running...';
    let endpoint = model === 'mavg' ? '/api/forecast/mavg' : '/api/forecast/exp';
    const params = new URLSearchParams({ storeId, itemId, days });
    const res = await fetch(endpoint + '?' + params.toString(), { method: 'POST' });
    if (!res.ok) {
      predictMsg.innerText = 'Failed: ' + await res.text();
      return;
    }
    const preds = await res.json();
    predictMsg.innerText = 'Done: ' + preds.length + ' days';
    // fetch combined history and predictions for chart
    await drawChart(storeId, itemId);
  });

  async function drawChart(storeId, itemId) {
    // fetch history
    // we'll fetch raw history via backend direct DB access using a simple endpoint: reuse /api/predictions for preds and we query history via server by adding a simple endpoint in future
    // For now we will call predictions and rely on DB for history via predictions + fetch from a simple server route not implemented; instead we'll combine with predictions and show preds only for demo

    // fetch predictions
    const res = await fetch(`/api/predictions?storeId=${storeId}&itemId=${itemId}`);
    const preds = await res.json();
    if (!Array.isArray(preds)) return;
    const labels = preds.map(p => p.date);
    const values = preds.map(p => p.predictedSales);

    const ctx = document.getElementById('chart').getContext('2d');
    if (chart) chart.destroy();
    chart = new Chart(ctx, {
      type: 'line',
      data: {
        labels,
        datasets: [{
          label: 'Predicted Sales',
          data: values,
          fill: false,
          tension: 0.2,
          borderWidth: 2
        }]
      },
      options: { responsive: true }
    });
  }
</script>
</body>
</html>
```

> Note: The UI above is intentionally simple. You can replace with React for a richer UX and fetch history from a new endpoint that returns actual historical sales + predictions combined. I kept the server-side minimal for ease-of-run.

---

# 11 — Docker files

`Dockerfile`

```dockerfile
FROM eclipse-temurin:17-jdk-jammy
WORKDIR /app
ARG JAR_FILE=target/ai-sales-predictor-0.0.1.jar
COPY ${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

`docker-compose.yml`

```yaml
version: "3.8"
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: ai_sales
    ports:
      - "3306:3306"
    volumes:
      - dbdata:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-ppassword"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build: .
    depends_on:
      db:
        condition: service_healthy
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/ai_sales?useSSL=false&allowPublicKeyRetrieval=true
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=password
    ports:
      - "8080:8080"

volumes:
  dbdata:
```

---

# 12 — How to run locally (quickstart)

1. Install Java 17+, Maven, Docker & Docker Compose (optional).
2. Build jar:

```bash
mvn clean package -DskipTests
```

3. Run with Docker Compose (recommended):

```bash
docker-compose up --build
```

This brings up MySQL and the Spring Boot app. Open `http://localhost:8080`.

If not using Docker, ensure MySQL is running and update `spring.datasource.url` to `jdbc:mysql://localhost:3306/ai_sales...` and run:

```bash
mvn spring-boot:run
```

4. Upload CSV via UI (index page) or via curl:

```bash
curl -F file=@sales.csv http://localhost:8080/api/upload
```

5. Run forecast from UI — select model and days.

---

# 13 — How to improve model (next steps / upgrades)

If you want *stronger AI models* later:

1. **Python ML microservice**

   * Build a Flask/FastAPI service that trains Prophet / XGBoost / LGBM / deep learning and exposes `/train` and `/predict`. Call it from `ForecastService` via HTTP (send prepared time series). This allows leveraging libraries like Prophet, tsfresh, XGBoost easily.

2. **Use Spark or DataFrame pipelines** for larger datasets.

3. **Neural Forecasting / LSTM / Transformer** for complex seasonality using PyTorch/Keras.

4. **Feature engineering**: day-of-week, holiday calendars (Indian holidays), promotions, price elasticity features, lag features, rolling stats.

5. **Hyperparameter tuning** and cross-validation to pick best performing model.

6. **Vector DB / embeddings** if you want to integrate product descriptions or text features using semantic search.

---

# 14 — Production concerns & monitoring

* Add **authentication & RBAC** (Spring Security + JWT) for shop owner and admins.
* Add **logging** & **metrics** (Prometheus + Grafana).
* Add **input validation**, **rate limits**, and **audit trails** for prediction changes.
* Persist trained model metadata, training timestamps, and model versions.
* Store backups of DB and prediction snapshots.

---

# 15 — Summary & offer

You now have:

* a complete, runnable Spring Boot project scaffold (full code above),
* a simple but effective forecasting engine in Java (moving average & exponential smoothing),
* UI to upload CSV and run predictions,
* Docker Compose to run MySQL + app with one command.

If you want, next I can:

1. Produce all files packaged into a Git repo (I will output any missing boilerplate files).
2. Replace the simple JS/Thymeleaf UI with a React dashboard + Chart.js and richer charts (historical vs predicted).
3. Add a Python ML microservice (FastAPI) implementing Prophet / XGBoost and show how to call it from Java.
4. Add authentication (Spring Security + JWT) and role-based UI.

Pick one of the next steps and I’ll generate the code/files for it right away.
