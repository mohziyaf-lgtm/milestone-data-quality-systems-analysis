# Milestone Data Quality, Systems Analysis and SQL Evidence

**N M Ziyaf**

Supply Chain Business Analyst

---

## Overview

I examined how shipment milestone events are recorded inside a CargoWise One freight forwarding environment, and found a data quality problem that originates in the design of the application rather than in the conduct of the operators who use it. This repository contains the SQL query library I wrote to measure the problem, the systems analysis document that interprets the evidence and sets out functional requirements for a fix, and a data flow diagram showing where the gap sits in the current system architecture.

This project is framed as a Systems Analyst would frame it, the application itself has a functional gap proven with data, not a process discipline problem that more operator effort would solve.

---

## The Real Problem

CargoWise One captures shipment milestones such as booking confirmed, customs cleared, and delivered to consignee as manually entered timestamps with no validation, no mandatory completion rule, and no chronological order check. Across the dataset I analysed, 21.9 percent of shipments are missing at least one milestone, the average entry delay is approximately 2.3 days, and 162 records exist where a later milestone was logged with an earlier timestamp than a milestone that must precede it, a chronological impossibility that no amount of operator care could explain. The overall data quality score across the full population is 38.8 percent.

---

## Why This Is a System Gap, Not a Process Gap

These figures are consistent and structural rather than tied to any individual operator's performance, which is exactly what a system design flaw looks like. The 162 impossible records are the clearest proof, the application accepted timestamps that violate the natural order of events because no validation rule exists to reject them at entry. No process improvement or operator training could have produced those records being correct, only a system level validation and exception layer can prevent them from being saved in the first place.

---

## Key Findings

| Measure | Result |
|---------|--------|
| Shipments missing at least one milestone | 21.9 percent |
| Average entry delay across milestones | Approximately 2.3 days |
| Overall data quality score | 38.8 percent |
| Shipments failing the defined quality threshold | 61.2 percent |
| Shipments with milestones entered by more than one operator | 35.3 percent |
| Chronologically impossible records | 162 |

---

## Files in This Repository

**Milestone_Data_Quality_Systems_Analysis.docx** and **.pdf**
The full systems analysis document. Covers an executive summary anchored in the SQL evidence, current system behaviour across capture mechanism, validation rules, and operator workflow, required system behaviour written as system level obligations, eight functional requirements with Must, Should, and Could priority and acceptance criteria, non functional requirements covering performance, auditability, and configurability, a data quality evidence section quoting the measured figures directly, a recommendation, and document control.

**Milestone_Data_Quality_SQL_Library.md**
Twelve SQL queries against a synthetic 4,000 shipment dataset, each with a prose explanation of the business question it answers and what the result reveals. Covers milestone completeness, entry delay distribution, operator level delay ranking, milestone type failure rates, month over month trend, chronological impossibility detection, multi operator handoff exposure, customer impact ranking, composite quality threshold scoring, and a final executive level monthly data quality score. Every figure in the systems analysis document traces back to one of these queries.

**milestone_dataflow.png**
A data flow diagram, not a swimlane process map, showing how milestone data currently moves from manual operator entry into CargoWise One and onward into shipment reporting and KPI dashboards with no validation layer at any point. A proposed Validation and Exception Engine is shown as the recommended addition, sitting between manual entry and the milestone record.

---

## Methodology Applied

SQL based data quality analysis using completeness, timeliness, and validity measures. Root cause reasoning that distinguishes a system design gap from an operator discipline gap using the evidence itself, specifically the chronologically impossible records that no individual behaviour could explain. Functional requirements writing in system obligation language rather than process instruction language. Data flow diagramming to show where a validation layer should sit in the existing architecture.

---

## Author

N M Ziyaf

Supply Chain Business Analyst, Sydney NSW

linkedin.com/in/ziyafmohamed
