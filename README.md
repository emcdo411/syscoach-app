🛡️ SysCoach: AI-Powered Windows System Assistant
A cross-functional project that combines PowerShell, R, and RShiny to scan, analyze, and intelligently respond to real-time Windows system performance and security data — built for developers, analysts, and non-technical users alike.

📌 Table of Contents

📖 Summary
🚀 Features
📦 Tech Stack
📚 Installed Packages
🧠 Why This Matters
🔧 PowerShell Audit Script
🧪 RShiny Dashboard Code
📁 File Structure
📝 UTF-8 Compatibility Fix
⚠️ Permissions Note
✅ Conclusion
🧭 Forking Workflow Diagram


📖 Summary
SysCoach is a secure, AI-guided Windows system auditor that provides real-time insights and control over system performance and security. It:

Scans system metrics using PowerShell to collect data on high CPU processes, services, Windows Defender status, and Spooler service status.
Exports results as structured, BOM-free JSON for reliable parsing.
Visualizes data in an interactive RShiny dashboard with tabs for CPU, Services, and Defender.
Executes safe commands like restarting the print spooler or enabling/disabling Wi-Fi adapters, with validation to ensure commands are viable.
Answers natural language queries (e.g., "What is svchost.exe?", "What are services?") with AI-style responses for educational insights.
Handles errors gracefully, displaying user-friendly messages for missing data or invalid commands.

Built for:

🧑‍💻 New developers learning PowerShell and R.
🧠 Data scientists analyzing system behavior.
👩‍💼 Non-technical users seeking intuitive system insights and control without terminal commands.


🚀 Features
✔️ PowerShell-based system health scan (CPU, Services, Defender, Spooler)✔️ BOM-free JSON export for reliable R parsing✔️ Smart file detection (auto-loads latest report from user-agnostic paths)✔️ Responsive RShiny dashboard with full-width tabs for CPU, Services, and Defender✔️ Safe command execution (e.g., restart spooler, enable/disable Wi-Fi) with Spooler validation✔️ AI-style query handler for questions about svchost, spooler, Wi-Fi, Defender, CPU, and services✔️ Graceful handling of missing data, invalid files, or command failures✔️ Dark-mode UI with Bootstrap 5 themes  

📦 Tech Stack


📚 Installed Packages


🧠 Why This Matters
Most system monitoring tools are either overly complex or locked behind GUI menus. SysCoach bridges this gap by:

Providing readable data visualizations for system metrics.
Enabling safe command execution with validation (e.g., checking Spooler service status).
Offering natural language answers to explain system components (e.g., "Which services are not running?").
Combining scripting and data science to enhance DevOps workflows.

SysCoach is more than a dashboard — it’s a smart command center for system observability and education.

🔧 PowerShell Audit Script
See: powershell/sentinel-audit.ps1
The script collects system data and saves it as a JSON file. It requires Administrator privileges for commands like Get-MpComputerStatus and Restart-Service. Key features:

Collects high CPU processes (>100 CPU time).
Lists all Windows services with status and start type.
Retrieves Windows Defender status.
Validates the Spooler service for command execution.
Uses BOM-free UTF-8 encoding.

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

Run as Administrator:
Start-Process powershell -Verb RunAs -ArgumentList "-File .\powershell\sentinel-audit.ps1"


🧪 RShiny Dashboard Code
See: R/app.R
The RShiny dashboard:

Auto-detects the latest JSON report using user-agnostic paths (%USERPROFILE%/Documents/SysCoach/output).
Displays tabs for High CPU Processes, Services, and Defender status.
Executes safe commands (e.g., restart spooler, enable/disable Wi-Fi) with validation for the Spooler service.
Responds to natural language queries (e.g., "What is svchost.exe?", "What are services?") with AI-style answers.
Handles missing or invalid data with user-friendly messages.

Copy and paste the full app.R code below:
library(shiny)
library(jsonlite)
library(DT)
library(bslib)

# 🔹 Smart loader for system reports
get_latest_report <- function() {
  search_paths <- c(
    file.path(Sys.getenv("USERPROFILE"), "Documents/SysCoach/output"),
    "output",
    "../output",
    file.path(getwd(), "output")
  )
  
  for (path in search_paths) {
    files <- list.files(path, pattern = "sys_report_.*\\.json$|sentinel_report_.*\\.json$", full.names = TRUE)
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

# 🔹 UI
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

# 🔹 Server
server <- function(input, output, session) {
  report_data <- reactiveVal(get_latest_report())
  
  observeEvent(input$cmd_run, {
    report_data(get_latest_report()) # Refresh data after commands
  })
  
  output$report_name <- renderText({
    req(report_data()$file)
    report_data()$file
  })
  
  # ---- Snapshot Outputs ----
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
  
  # ---- Safe Command Translator ----
  observeEvent(input$cmd_run, {
    cmd <- tolower(trimws(input$cmd_input))
    # Dynamically get Wi-Fi adapter name
    wifi_adapter <- tryCatch({
      output <- system2("powershell", args = c("-Command", "Get-NetAdapter | Where-Object { $_.Name -like '*Wi-Fi*' -or $_.Name -like '*Wireless*' } | Select-Object -ExpandProperty Name -First 1"), stdout = TRUE, stderr = TRUE, timeout = 5)
      if (length(output) == 0 || is.null(output)) {
        "Wi-Fi" # Fallback
      } else {
        trimws(output[1])
      }
    }, warning = function(w) {
      message("Wi-Fi adapter warning: ", w$message)
      "Wi-Fi"
    }, error = function(e) {
      message("Wi-Fi adapter detection error: ", e$message)
      "Wi-Fi"
    })
    
    allowed <- list(
      "restart spooler" = "Restart-Service -Name 'Spooler' -Force",
      "enable wifi" = sprintf("Enable-NetAdapter -Name '%s' -Confirm:$false", wifi_adapter),
      "disable wifi" = sprintf("Disable-NetAdapter -Name '%s' -Confirm:$false", wifi_adapter)
    )
    
    if (nzchar(cmd) && cmd %in% names(allowed)) {
      # Check Spooler service status before restarting
      if (cmd == "restart spooler") {
        spooler_check <- tryCatch({
          output <- system2("powershell", args = c("-Command", "Get-Service -Name 'Spooler' | Select-Object -ExpandProperty Status"), stdout = TRUE, stderr = TRUE, timeout = 5)
          if (length(output) == 0 || is.null(output)) {
            "NotFound"
          } else {
            trimws(output[1])
          }
        }, warning = function(w) {
          message("Spooler check warning: ", w$message)
          "Error"
        }, error = function(e) {
          message("Spooler check error: ", e$message)
          "Error"
        })
        if (spooler_check == "NotFound" || spooler_check == "Error") {
          output$cmd_result <- renderText({
            "❌ Spooler service not found or inaccessible. Run PowerShell as Administrator."
          })
          return()
        }
      }
      
      output$cmd_result <- renderText({
        result <- tryCatch({
          output <- system2("powershell", args = c("-Command", allowed[[cmd]]), stdout = TRUE, stderr = TRUE, timeout = 5)
          if (length(output) == 0) {
            "✅ Command executed successfully."
          } else {
            paste(output, collapse = "\n")
          }
        }, warning = function(w) {
          message("Command warning: ", w$message)
          paste("⚠️ Command warning: ", w$message)
        }, error = function(e) {
          message("Command error: ", e$message)
          paste("❌ Command failed: ", e$message)
        })
        result
      })
    } else {
      output$cmd_result <- renderText({
        "❌ Unknown or unsafe command. Try: restart spooler, enable wifi, disable wifi."
      })
    }
  })
  
  # ---- AI Educational Companion ----
  observeEvent(input$ask_submit, {
    question <- tolower(trimws(input$ask_input))
    if (!nzchar(question)) {
      output$ask_response <- renderText({ "⚠️ Please enter a valid question." })
      return()
    }
    response <- switch(TRUE,
      grepl("svchost", question) ~ "'svchost.exe' is a generic host process for services running from DLLs, used by many Windows processes.",
      grepl("spooler", question) ~ "The print spooler manages print jobs. Restarting it can resolve printer issues.",
      grepl("wifi|wi-fi", question) ~ "Wi-Fi adapters connect to wireless networks. Use 'enable wifi' or 'disable wifi' to control them.",
      grepl("defender|antivirus", question) ~ "Windows Defender is Microsoft's built-in antivirus, protecting against malware and threats.",
      grepl("cpu|processor", question) ~ "CPU usage reflects system performance. High usage may indicate heavy processes or bottlenecks.",
      grepl("service|services", question) ~ "Windows services are background processes managed by the Service Control Manager, like print spooler or networking services.",
      TRUE ~ "🧠 Sorry, I don’t have an answer for that yet. Try asking about svchost, spooler, wifi, defender, cpu, or services."
    )
    output$ask_response <- renderText({ as.character(response) })
  })
}

# 🔹 Run the app
shinyApp(ui = ui, server = server)


📁 File Structure
syscoach/
├── powershell/
│   └── sentinel-audit.ps1
├── R/
│   └── app.R
├── output/
│   └── sys_report_<timestamp>.json
├── README.md


📝 UTF-8 Compatibility Fix
PowerShell’s default output may include a UTF-8 BOM, causing JSON parsing errors in R. The script uses BOM-free UTF-8 encoding:
$utf8NoBom = [System.Text.UTF8Encoding]::new($false)
[System.IO.File]::WriteAllText($finalPath, $json, $utf8NoBom)


⚠️ Permissions Note
Commands like restart spooler and data collection for Services/Defender require Administrator privileges. Run both PowerShell and RStudio as Administrator:

PowerShell: Start-Process powershell -Verb RunAs -ArgumentList "-File .\powershell\sentinel-audit.ps1"
RStudio: Right-click RStudio → “Run as administrator”

Verify the Spooler service:
Get-Service -Name Spooler

If the Spooler service is missing or inaccessible, check system configuration or permissions.

✅ Conclusion
SysCoach merges system diagnostics, safe command execution, data visualization, and AI-driven feedback into a user-friendly platform. It’s an evolving tool for AI-assisted system observability and education. Fork it, run it, and extend it with new commands or queries.

🧭 Forking Workflow Diagram
flowchart LR
    A[Original Repo: main] -->|Fork| B[Your Fork: main]
    B -->|Create Branch| C[feature/syscoach]
    C -->|Commit| D[Local Commits]
    D -->|Push| E[Pull Request]
    E -->|Merge| F[Main Updated]
