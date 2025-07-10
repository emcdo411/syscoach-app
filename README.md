# 🛡️ SysCoach: AI-Powered Windows System Assistant

A cross-functional project that combines PowerShell, R, and RShiny to scan, analyze, and intelligently respond to real-time Windows system performance and security data — built for developers, analysts, and non-technical users alike.

---

## 📌 Table of Contents

* [📖 Summary](#📖-summary)
* [🚀 Features](#🚀-features)
* [📦 Tech Stack](#📦-tech-stack)
* [📚 Installed Packages](#📚-installed-packages)
* [🧠 Why This Matters](#🧠-why-this-matters)
* [🔧 PowerShell Audit Script](#🔧-powershell-audit-script)
* [🧪 RShiny Dashboard Code](#🧪-rshiny-dashboard-code)
* [📁 File Structure](#📁-file-structure)
* [📝 UTF-8 Compatibility Fix](#📝-utf-8-compatibility-fix)
* [⚠️ Permissions Note](#⚠️-permissions-note)
* [✅ Conclusion](#✅-conclusion)
* [🧭 Forking Workflow Diagram](#🧭-forking-workflow-diagram)

---

## 📖 Summary

**SysCoach** is a secure, AI-guided Windows system auditor that provides real-time insights and control over system performance and security. It:

* Scans system metrics using PowerShell to collect data on high CPU processes, services, Windows Defender status, and Spooler service status.
* Exports results as structured, BOM-free JSON for reliable parsing.
* Visualizes data in an interactive RShiny dashboard with tabs for CPU, Services, and Defender.
* Executes safe commands like restarting the print spooler or enabling/disabling Wi-Fi adapters, with validation.
* Answers natural language queries with AI-style responses for educational insights.
* Handles errors gracefully with user-friendly messages.

For:

* 🧑‍💻 New developers learning PowerShell and R
* 🧠 Data scientists analyzing system behavior
* 👩‍💼 Non-technical users needing intuitive system insights

---

## 🚀 Features

✔️ PowerShell-based system health scan (CPU, Services, Defender, Spooler)
✔️ BOM-free JSON export for reliable R parsing
✔️ Smart file detection across user paths
✔️ Responsive RShiny dashboard with themed tabs
✔️ Safe command execution with validation (restart spooler, enable/disable Wi-Fi)
✔️ AI-style query handler for system terms and usage
✔️ Graceful error handling for all stages
✔️ Bootstrap 5 dark mode styling

---

## 📦 Tech Stack

![PowerShell](https://img.shields.io/badge/PowerShell-Windows-blue?logo=powershell\&logoColor=white)
![R](https://img.shields.io/badge/R-Language-276DC3?logo=r\&logoColor=white)
![RShiny](https://img.shields.io/badge/Shiny-Interactive%20UI-brightgreen?logo=rstudio\&logoColor=white)
![JSON](https://img.shields.io/badge/JSON-Structured%20Data-lightgrey?logo=json\&logoColor=black)

---

## 📚 Installed Packages

![jsonlite](https://img.shields.io/badge/jsonlite-JSON%20Parser-lightblue)
![DT](https://img.shields.io/badge/DT-DataTables-blue)
![bslib](https://img.shields.io/badge/bslib-Bootstrap%20Themes-purple)
![shiny](https://img.shields.io/badge/shiny-Web%20Framework-orange)

---

## 🧠 Why This Matters

Most system monitoring tools are either overly complex or locked behind GUI menus. SysCoach bridges this gap by:

* Providing readable data visualizations for system metrics.
* Enabling safe command execution with validation.
* Offering natural language answers to explain system components.
* Combining scripting and data science to enhance DevOps workflows.

SysCoach is more than a dashboard — it’s a smart command center.

---

## 🔧 PowerShell Audit Script

See: `powershell/sentinel-audit.ps1`

```powershell
$homeFolder = [Environment]::GetFolderPath("MyDocuments")
$projectRoot = Join-Path $homeFolder "SysCoach"
$outputPath = Join-Path $projectRoot "output"

if (!(Test-Path $outputPath)) {
    New-Item -Path $outputPath -ItemType Directory -Force | Out-Null
}

$report = @{}

try {
    $report["HighCPUProcesses"] = Get-Process |
        Where-Object { $_.CPU -gt 100 } |
        Select-Object Name, Id, CPU -ErrorAction Stop
} catch {
    $report["HighCPUProcesses"] = @{ Error = "Failed to retrieve process data: $_" }
}

try {
    $report["Services"] = Get-Service |
        Select-Object Name, Status, StartType -ErrorAction Stop
} catch {
    $report["Services"] = @{ Error = "Failed to retrieve services data: $_" }
}

try {
    $defender = Get-MpComputerStatus -ErrorAction Stop
    $report["Defender"] = $defender | Select-Object AMServiceEnabled, RealTimeProtectionEnabled, AntispywareEnabled
} catch {
    $report["Defender"] = @{ Error = "Defender data not available: $_" }
}

try {
    $spooler = Get-Service -Name 'Spooler' -ErrorAction Stop
    $report["SpoolerStatus"] = @{
        Name = $spooler.Name
        Status = $spooler.Status.ToString()
        StartType = $spooler.StartType.ToString()
    }
} catch {
    $report["SpoolerStatus"] = @{ Error = "Spooler service not found or inaccessible: $_" }
}

$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$fileName = "sys_report_$timestamp.json"
$finalPath = Join-Path $outputPath $fileName

try {
    $json = $report | ConvertTo-Json -Depth 5 -ErrorAction Stop
    $utf8NoBom = [System.Text.UTF8Encoding]::new($false)
    [System.IO.File]::WriteAllText($finalPath, $json, $utf8NoBom)
} catch {
    Write-Error "Failed to write JSON file: $_"
}
```

Run as Administrator:

```powershell
Start-Process powershell -Verb RunAs -ArgumentList "-File .\powershell\sentinel-audit.ps1"
```

---

## 🧪 RShiny Dashboard Code

See: `R/app.R`

```r
library(shiny)
library(jsonlite)
library(DT)
library(bslib)

# Smart loader for system reports
get_latest_report <- function() {
  search_paths <- c(
    file.path(Sys.getenv("USERPROFILE"), "Documents/SysCoach/output"),
    "output",
    "../output",
    file.path(getwd(), "output")
  )

  for (path in search_paths) {
    files <- list.files(path, pattern = "sys_report_.*\.json$|sentinel_report_.*\.json$", full.names = TRUE)
    if (length(files) > 0) {
      latest <- files[which.max(file.info(files)$mtime)]
      data <- tryCatch({
        jsonlite::fromJSON(latest, simplifyVector = TRUE, simplifyDataFrame = TRUE)
      }, warning = function(w) {
        message("JSON warning for ", latest, ": ", w$message)
        NULL
      }, error = function(e) {
        message("JSON parsing error for ", latest, ": ", e$message)
        NULL
      })
      return(list(file = basename(latest), data = data))
    }
  }

  message("No JSON files found in search paths: ", paste(search_paths, collapse = ", "))
  return(list(file = "❌ No report found", data = NULL))
}

# UI
ui <- fluidPage(
  theme = bs_theme(version = 5, bootswatch = "darkly", base_font = font_google("Roboto")),
  titlePanel("🧠 SysCoach - AI System Companion & Command Center"),

  tabsetPanel(
    tabPanel("📊 System Snapshot",
      fluidRow(
        column(12,
          strong("📁 Report Loaded:"),
          textOutput("report_name"),
          br(),
          tabsetPanel(
            tabPanel("🔥 High CPU", DTOutput("cpu")),
            tabPanel("🔧 Services", DTOutput("services")),
            tabPanel("🛡️ Defender", DTOutput("defender"))
          )
        )
      )
    ),
    tabPanel("💻 Run Safe Commands",
      textInput("cmd_input", "Enter safe action:", placeholder = "e.g., restart spooler"),
      actionButton("cmd_run", "▶️ Run"),
      verbatimTextOutput("cmd_result")
    ),
    tabPanel("🧠 Ask SysCoach",
      textInput("ask_input", "Ask a system question:", placeholder = "e.g., What is svchost.exe?"),
      actionButton("ask_submit", "💬 Ask"),
      verbatimTextOutput("ask_response")
    )
  )
)

# Server
server <- function(input, output, session) {
  report_data <- reactiveVal(get_latest_report())

  observeEvent(input$cmd_run, {
    report_data(get_latest_report())
  })

  output$report_name <- renderText({
    req(report_data()$file)
    report_data()$file
  })

  output$cpu <- renderDT({
    sysdata <- report_data()$data
    if (is.null(sysdata) || !is.data.frame(sysdata$HighCPUProcesses)) {
      datatable(data.frame(Message = "No CPU data available"))
    } else {
      datatable(sysdata$HighCPUProcesses)
    }
  })

  output$services <- renderDT({
    sysdata <- report_data()$data
    if (is.null(sysdata) || !is.data.frame(sysdata$Services)) {
      datatable(data.frame(Message = "No services data available"))
    } else {
      datatable(sysdata$Services)
    }
  })

  output$defender <- renderDT({
    sysdata <- report_data()$data
    if (is.null(sysdata) || !is.list(sysdata$Defender)) {
      datatable(data.frame(Message = "No Defender data available"))
    } else {
      datatable(as.data.frame(sysdata$Defender))
    }
  })

  observeEvent(input$cmd_run, {
    cmd <- tolower(trimws(input$cmd_input))
    wifi_adapter <- tryCatch({
      output <- system2("powershell", args = c("-Command", "Get-NetAdapter | Where-Object { $_.Name -like '*Wi-Fi*' -or $_.Name -like '*Wireless*' } | Select-Object -ExpandProperty Name -First 1"), stdout = TRUE, stderr = TRUE, timeout = 5)
      if (length(output) == 0 || is.null(output)) {
        "Wi-Fi"
      } else {
        trimws(output[1])
      }
    }, error = function(e) "Wi-Fi")

    allowed <- list(
      "restart spooler" = "Restart-Service -Name 'Spooler' -Force",
      "enable wifi" = sprintf("Enable-NetAdapter -Name '%s' -Confirm:$false", wifi_adapter),
      "disable wifi" = sprintf("Disable-NetAdapter -Name '%s' -Confirm:$false", wifi_adapter)
    )

    output$cmd_result <- renderText({
      if (nzchar(cmd) && cmd %in% names(allowed)) {
        tryCatch({
          output <- system2("powershell", args = c("-Command", allowed[[cmd]]), stdout = TRUE, stderr = TRUE, timeout = 5)
          paste(output, collapse = "
")
        }, error = function(e) paste("❌ Command failed:", e$message))
      } else {
        "❌ Unknown or unsafe command."
      }
    })
  })

  observeEvent(input$ask_submit, {
    question <- tolower(trimws(input$ask_input))
    response <- switch(TRUE,
      grepl("svchost", question) ~ "'svchost.exe' is a generic host process for Windows services.",
      grepl("spooler", question) ~ "The print spooler manages print jobs. Restarting it can fix printer issues.",
      grepl("wifi|wi-fi", question) ~ "Wi-Fi adapters allow wireless connectivity. You can enable or disable them via PowerShell.",
      grepl("defender", question) ~ "Windows Defender is built-in antivirus protection.",
      grepl("cpu", question) ~ "CPU usage measures how much processing power is being used by active processes.",
      grepl("services", question) ~ "Windows Services are background processes started by the OS.",
      TRUE ~ "🧠 Sorry, I don’t have an answer for that yet."
    )
    output$ask_response <- renderText({ response })
  })
}

# Run the app
shinyApp(ui = ui, server = server)
```

---

## 📁 File Structure

```
syscoach/
├── powershell/
│   └── sentinel-audit.ps1
├── R/
│   └── app.R
├── output/
│   └── sys_report_<timestamp>.json
├── README.md
```

---

## 📝 UTF-8 Compatibility Fix

To avoid R JSON parsing issues from BOM encoding:

```powershell
$utf8NoBom = [System.Text.UTF8Encoding]::new($false)
[System.IO.File]::WriteAllText($finalPath, $json, $utf8NoBom)
```

---

## ⚠️ Permissions Note

Commands like `Restart-Service` and `Get-MpComputerStatus` require elevated privileges.

* Run PowerShell scripts as Administrator:

```powershell
Start-Process powershell -Verb RunAs -ArgumentList "-File .\powershell\sentinel-audit.ps1"
```

* Run RStudio as Administrator if needed

To verify:

```powershell
Get-Service -Name Spooler
```

---

## ✅ Conclusion

**SysCoach** blends system diagnostics, safe command automation, AI-style feedback, and modern dashboards. It’s built for real-world problem-solving and scalable DevOps extensions.

Use it. Fork it. Improve it.

---

## 🧭 Forking Workflow Diagram

```mermaid
flowchart LR
    A[Original Repo: main] -->|Fork| B[Your Fork: main]
    B -->|Create Branch| C[feature/syscoach]
    C -->|Commit| D[Local Commits]
    D -->|Push| E[Pull Request]
    E -->|Merge| F[Main Updated]
```
