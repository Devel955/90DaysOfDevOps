# Day 20 – Bash Scripting Challenge  
## Log Analyzer and Report Generator

This project is a real-world DevOps style Bash scripting challenge focused on log analysis and automation.

## 🚀 Project Overview

As a system administrator, analyzing log files is a daily responsibility.  
This script automates log analysis by extracting errors, identifying critical events, and generating a structured summary report.

The goal of this project is to practice:

- Input validation
- Text processing using grep, awk, sed
- Sorting and counting log data
- Report generation using Bash scripting

---

## ⚙️ Features

✅ Accepts log file as a command-line argument  
✅ Validates file existence  
✅ Counts total `ERROR` and `Failed` entries  
✅ Extracts `CRITICAL` events with line numbers  
✅ Identifies Top 5 most frequent error messages  
✅ Generates a summary report: `log_report_<date>.txt`  

---

## 🛠 Tools & Commands Used

- `grep`
- `awk`
- `sed`
- `sort`
- `uniq`
- `wc`
- `date`
- Output redirection (`>`)

---

## ▶️ How to Run

```bash
chmod +x log_analyzer.sh
./log_analyzer.sh sample.log

Output
The script generates a report file:
log_report_<YYYY-MM-DD>.txt
The report includes:

Date of analysis
Log file name
Total lines processed
Total error count
Top 5 error messages
List of critical events
