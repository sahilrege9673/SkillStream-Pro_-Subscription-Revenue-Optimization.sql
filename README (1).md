# 📊 SkillStream Pro: Subscription Revenue Optimization with SQL

![Project Status](https://img.shields.io/badge/Status-Complete-success)
![SQL](https://img.shields.io/badge/SQL-PostgreSQL-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Intermediate%20to%20Advanced-orange)
![Business Domain](https://img.shields.io/badge/Domain-SaaS%20Analytics-purple)

## 🎯 Project Overview

A comprehensive SQL and excel analytics project solving a real-world SaaS business challenge: **SkillStream Pro**, a digital learning platform with 50,000 subscribers, experiencing an **18% revenue per user decline** despite growing user base.

This project demonstrates advanced SQL capabilities for subscription revenue optimization, churn prediction, and customer lifetime value analysis—skills directly applicable to roles at companies like Netflix, Spotify, LinkedIn, Salesforce, and any subscription-based business.

---

## 🚨 The Business Problem

**Critical Issue:** Revenue per user (RPU) declined 18% in Q1 2026 while subscriber count increased 12%

**Board Meeting Pressure:** CFO needs data-backed answers before Q2 2026 board presentation

**Root Cause Hypotheses:**
- High-value customer churn concentration
- Users downgrading from premium to basic plans  
- New subscribers choosing lower-tier plans
- Inefficient pricing structure
- Low-quality customer acquisition channels

---

## 💡 Key Insights Delivered

### 1️⃣ Churn Prediction
- **80% churn probability** after 30+ days of inactivity
- Payment failure patterns strongly correlate with cancellation risk
- Early intervention can save $90K in annual recurring revenue

### 2️⃣ Retention Drivers  
- Users completing **2+ courses in first 7 days** have **3x retention rate**
- Optimal activation metric identified for onboarding optimization
- Engagement threshold: 10+ total courses completed drives 45% upgrade rate

### 3️⃣ Acquisition Channel ROI
- **Referral & LinkedIn channels** produce 2.5x higher customer LTV than Facebook/Google Ads
- Marketing budget reallocation could improve average LTV by 25%
- Low-performing channels identified for optimization or elimination

### 4️⃣ Pricing Strategy Validation
- Annual billing shows stronger retention but underperforms industry discount standards
- Upgrade/downgrade trends reveal pricing elasticity by customer segment
- Optimal price points identified through conversion and retention analysis

### 5️⃣ Revenue Recovery Opportunities
- **Payment failure recovery** improvement: 30% → 65% adds $15K MRR
- Targeted retention campaigns for high-risk, high-value users: 300% ROI
- Predictive churn alerting system reduces churn by 2-3 percentage points

---

## 📚 SQL Questions Overview

### **LEVEL 1: Basic Business SQL** (Questions 1-5)
Foundation queries for core metrics
- Total revenue by month
- Active subscriptions by plan type  
- Geographic subscriber distribution
- Average subscription duration
- Conversion funnel analysis (signups without subscriptions)

### **LEVEL 2: Intermediate Analytics** (Questions 6-15)  
Business logic with window functions and CTEs
- Month-over-month revenue growth (LAG function)
- User downgrade detection (self-join analysis)
- Cohort-based churn rate calculation
- Revenue per user by acquisition channel (ROW_NUMBER ranking)
- Payment failure pattern analysis
- Customer lifetime value calculation
- Churn risk identification (30-day inactivity threshold)
- Engagement comparison: churned vs retained users
- Upgrade/downgrade trend analysis (LEAD/LAG functions)
- Optimal pricing point evaluation

### **LEVEL 3: Advanced Strategic Insights** (Questions 16-21)
Predictive and prescriptive analytics
- Acquisition channel LTV comparison (6-month horizon)
- Early engagement behaviors predicting 12-month retention
- Course completion impact on upgrade probability
- Trial extension revenue impact modeling
- Risk-scored user segmentation for retention campaigns
- Executive dashboard KPI snapshot


