---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered task management"
  - "create user management with anomaly detection"
  - "build admin dashboard with AI analytics"
  - "integrate ticket classification system"
  - "add burnout prediction to user management"
  - "configure AI risk detection for users"
  - "develop enterprise management with ML insights"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with AI-driven insights. It provides role-based access control, task tracking with Kanban boards, support ticket management, and AI features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

**Key Components:**
- **Frontend**: React.js dashboard for users and admins
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI microservice with scikit-learn and River for online learning
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
```

## Architecture

### Backend API Structure

The Node.js backend follows MVC pattern:

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

exports.createUser = async (req, res) => {
  try {
    const { name, email, role, department } = req.body;
    
    const user = await User.create({
      name,
      email,
      role, // 'admin' or 'user'
      department,
      status: 'active',
      createdAt: new Date()
    });
    
    res.status(201).json({ success: true, data: user });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};

exports.getUserAnalytics = async (req, res) => {
  try {
    const { userId } = req.params;
    
    // Fetch user tasks and activity
    const tasks = await Task.find({ assignedTo: userId });
    const tickets = await Ticket.find({ userId });
    
    // Call ML service for AI insights
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/analyze/user`, {
      userId,
      tasks: tasks.length,
      completedTasks: tasks.filter(t => t.status === 'done').length,
      avgTaskDuration: calculateAvgDuration(tasks),
      ticketCount: tickets.length
    });
    
    res.json({
      success: true,
      analytics: mlResponse.data
    });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};
```

### Task Management API

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      priority, // 'low', 'medium', 'high', 'critical'
      status: 'todo',
      dueDate: new Date(dueDate),
      createdBy: req.user.id,
      timeTracked: 0
    });
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { taskId } = req.params;
    const { status } = req.body; // 'todo', 'in_progress', 'done'
    
    const task = await Task.findByIdAndUpdate(
      taskId,
      { 
        status,
        updatedAt: new Date(),
        ...(status === 'done' && { completedAt: new Date() })
      },
      { new: true }
    );
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const { taskId } = req.params;
    const { timeSpent } = req.body; // in seconds
    
    const task = await Task.findByIdAndUpdate(
      taskId,
      { $inc: { timeTracked: timeSpent } },
      { new: true }
    );
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};
```

## ML Service API

### AI-Powered Ticket Classification

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketRequest(BaseModel):
    title: str
    description: str
    userId: str

class TicketClassification(BaseModel):
    category: str
    priority: str
    confidence: float
    assignedDepartment: str

@app.post("/classify/ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketRequest):
    try:
        # Load trained model
        vectorizer = joblib.load(f"{MODEL_PATH}/ticket_vectorizer.pkl")
        classifier = joblib.load(f"{MODEL_PATH}/ticket_classifier.pkl")
        
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        # Transform and predict
        features = vectorizer.transform([text])
        category = classifier.predict(features)[0]
        confidence = max(classifier.predict_proba(features)[0])
        
        # Determine priority based on keywords
        priority = determine_priority(text)
        department = map_category_to_department(category)
        
        return TicketClassification(
            category=category,
            priority=priority,
            confidence=float(confidence),
            assignedDepartment=department
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def determine_priority(text: str) -> str:
    urgent_keywords = ['urgent', 'critical', 'asap', 'emergency', 'down', 'crash']
    high_keywords = ['important', 'soon', 'needed', 'issue']
    
    text_lower = text.lower()
    if any(keyword in text_lower for keyword in urgent_keywords):
        return 'critical'
    elif any(keyword in text_lower for keyword in high_keywords):
        return 'high'
    else:
        return 'medium'

def map_category_to_department(category: str) -> str:
    mapping = {
        'technical': 'IT',
        'billing': 'Finance',
        'account': 'Customer Service',
        'feature_request': 'Product',
        'bug': 'Engineering'
    }
    return mapping.get(category, 'General Support')
```

### Risk Detection

```python
# ml-service/risk_detection.py
from pydantic import BaseModel
from typing import List
import numpy as np

class UserBehavior(BaseModel):
    userId: str
    loginTimes: List[int]  # hours of day
    failedLogins: int
    dataAccessed: int
    unusualActivities: int
    workingHours: List[int]

class RiskScore(BaseModel):
    userId: str
    riskLevel: str  # 'low', 'medium', 'high'
    score: float
    anomalies: List[str]
    recommendations: List[str]

@app.post("/analyze/risk", response_model=RiskScore)
async def detect_risk(behavior: UserBehavior):
    try:
        anomalies = []
        score = 0.0
        
        # Check failed logins
        if behavior.failedLogins > 5:
            anomalies.append("Excessive failed login attempts")
            score += 0.3
        
        # Check unusual login times
        unusual_hours = [h for h in behavior.loginTimes if h < 6 or h > 22]
        if len(unusual_hours) > 3:
            anomalies.append("Login attempts outside normal hours")
            score += 0.2
        
        # Check data access patterns
        if behavior.dataAccessed > 1000:
            anomalies.append("Unusually high data access")
            score += 0.25
        
        # Check unusual activities
        if behavior.unusualActivities > 0:
            anomalies.append(f"{behavior.unusualActivities} unusual activities detected")
            score += behavior.unusualActivities * 0.1
        
        # Determine risk level
        if score >= 0.7:
            risk_level = 'high'
        elif score >= 0.4:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        recommendations = generate_recommendations(risk_level, anomalies)
        
        return RiskScore(
            userId=behavior.userId,
            riskLevel=risk_level,
            score=min(score, 1.0),
            anomalies=anomalies,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def generate_recommendations(risk_level: str, anomalies: List[str]) -> List[str]:
    recommendations = []
    
    if risk_level == 'high':
        recommendations.append("Require immediate password reset")
        recommendations.append("Enable two-factor authentication")
        recommendations.append("Review user access permissions")
    elif risk_level == 'medium':
        recommendations.append("Monitor user activity closely")
        recommendations.append("Send security notification to user")
    
    if "failed login" in str(anomalies).lower():
        recommendations.append("Check for potential brute force attack")
    
    return recommendations
```

### Burnout Detection

```python
# ml-service/burnout_detection.py
from pydantic import BaseModel
from datetime import datetime, timedelta

class UserWorkload(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    avgWorkHoursPerDay: float
    overtimeHours: float
    missedDeadlines: int
    lastVacationDays: int

class BurnoutAnalysis(BaseModel):
    userId: str
    burnoutRisk: str  # 'low', 'moderate', 'high'
    score: float
    factors: List[str]
    suggestions: List[str]

@app.post("/analyze/burnout", response_model=BurnoutAnalysis)
async def detect_burnout(workload: UserWorkload):
    try:
        factors = []
        score = 0.0
        
        # Check workload
        task_completion_rate = workload.tasksCompleted / max(workload.tasksAssigned, 1)
        if task_completion_rate < 0.7:
            factors.append("Low task completion rate")
            score += 0.25
        
        # Check work hours
        if workload.avgWorkHoursPerDay > 10:
            factors.append("Excessive daily work hours")
            score += 0.3
        
        # Check overtime
        if workload.overtimeHours > 20:
            factors.append("High overtime hours")
            score += 0.2
        
        # Check missed deadlines
        if workload.missedDeadlines > 3:
            factors.append("Frequent missed deadlines")
            score += 0.15
        
        # Check vacation time
        if workload.lastVacationDays > 180:
            factors.append("No vacation in 6+ months")
            score += 0.1
        
        # Determine burnout risk
        if score >= 0.6:
            burnout_risk = 'high'
        elif score >= 0.3:
            burnout_risk = 'moderate'
        else:
            burnout_risk = 'low'
        
        suggestions = generate_burnout_suggestions(burnout_risk, factors)
        
        return BurnoutAnalysis(
            userId=workload.userId,
            burnoutRisk=burnout_risk,
            score=min(score, 1.0),
            factors=factors,
            suggestions=suggestions
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def generate_burnout_suggestions(risk: str, factors: List[str]) -> List[str]:
    suggestions = []
    
    if risk == 'high':
        suggestions.append("Recommend immediate workload reduction")
        suggestions.append("Schedule mandatory time off")
        suggestions.append("Arrange meeting with manager")
    elif risk == 'moderate':
        suggestions.append("Redistribute some tasks to team members")
        suggestions.append("Encourage regular breaks")
    
    if "overtime" in str(factors).lower():
        suggestions.append("Limit overtime to essential tasks only")
    
    if "vacation" in str(factors).lower():
        suggestions.append("Encourage taking vacation days")
    
    return suggestions
```

## Frontend Integration

### React Component for Task Dashboard

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = ({ userId, isAdmin }) => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });
  const [loading, setLoading] = useState(true);
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const endpoint = isAdmin 
        ? `${API_URL}/api/tasks/all`
        : `${API_URL}/api/tasks/user/${userId}`;
      
      const response = await axios.get(endpoint, {
        headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
      });
      
      const grouped = response.data.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], in_progress: [], done: [] });
      
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${localStorage.getItem('token')}` } }
      );
      fetchTasks(); // Refresh tasks
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const renderColumn = (status, title) => (
    <div className="task-column">
      <h3>{title} ({tasks[status].length})</h3>
      {tasks[status].map(task => (
        <div 
          key={task._id} 
          className={`task-card priority-${task.priority}`}
          draggable
          onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}
        >
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className="priority-badge">{task.priority}</span>
          <span className="time-tracked">{formatTime(task.timeTracked)}</span>
        </div>
      ))}
    </div>
  );

  const formatTime = (seconds) => {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    return `${hours}h ${minutes}m`;
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('in_progress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard Component

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const [riskData, burnoutData] = await Promise.all([
        fetchRiskAnalysis(),
        fetchBurnoutAnalysis()
      ]);
      
      setAnalytics({ risk: riskData, burnout: burnoutData });
      setLoading(false);
    } catch (error) {
      console.error('Error fetching analytics:', error);
      setLoading(false);
    }
  };

  const fetchRiskAnalysis = async () => {
    const response = await axios.post(`${ML_API_URL}/analyze/risk`, {
      userId,
      loginTimes: [9, 10, 14, 15, 18],
      failedLogins: 2,
      dataAccessed: 450,
      unusualActivities: 0,
      workingHours: [9, 10, 11, 12, 13, 14, 15, 16, 17]
    });
    return response.data;
  };

  const fetchBurnoutAnalysis = async () => {
    const response = await axios.post(`${ML_API_URL}/analyze/burnout`, {
      userId,
      tasksAssigned: 25,
      tasksCompleted: 18,
      avgWorkHoursPerDay: 8.5,
      overtimeHours: 12,
      missedDeadlines: 2,
      lastVacationDays: 90
    });
    return response.data;
  };

  if (loading) return <div>Loading AI insights...</div>;

  return (
    <div className="ai-analytics">
      <div className="risk-analysis card">
        <h3>Security Risk Analysis</h3>
        <div className={`risk-level ${analytics.risk.riskLevel}`}>
          {analytics.risk.riskLevel.toUpperCase()}
        </div>
        <p>Risk Score: {(analytics.risk.score * 100).toFixed(1)}%</p>
        
        {analytics.risk.anomalies.length > 0 && (
          <div className="anomalies">
            <h4>Detected Anomalies:</h4>
            <ul>
              {analytics.risk.anomalies.map((anomaly, i) => (
                <li key={i}>{anomaly}</li>
              ))}
            </ul>
          </div>
        )}
        
        <div className="recommendations">
          <h4>Recommendations:</h4>
          <ul>
            {analytics.risk.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      </div>

      <div className="burnout-analysis card">
        <h3>Burnout Risk Assessment</h3>
        <div className={`burnout-level ${analytics.burnout.burnoutRisk}`}>
          {analytics.burnout.burnoutRisk.toUpperCase()}
        </div>
        <p>Burnout Score: {(analytics.burnout.score * 100).toFixed(1)}%</p>
        
        {analytics.burnout.factors.length > 0 && (
          <div className="factors">
            <h4>Contributing Factors:</h4>
            <ul>
              {analytics.burnout.factors.map((factor, i) => (
                <li key={i}>{factor}</li>
              ))}
            </ul>
          </div>
        )}
        
        <div className="suggestions">
          <h4>Suggestions:</h4>
          <ul>
            {analytics.burnout.suggestions.map((suggestion, i) => (
              <li key={i}>{suggestion}</li>
            ))}
          </ul>
        </div>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ 
        success: false, 
        error: 'No token provided' 
      });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);
    
    if (!user) {
      return res.status(401).json({ 
        success: false, 
        error: 'User not found' 
      });
    }
    
    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ 
      success: false, 
      error: 'Invalid token' 
    });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: 'Insufficient permissions'
      });
    }
    next();
  };
};
```

## Common Patterns

### Create Support Ticket with AI Classification

```javascript
// Backend endpoint
app.post('/api/tickets', authenticate, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify/ticket`, {
      title,
      description,
      userId: req.user.id
    });
    
    const ticket = await Ticket.create({
      title,
      description,
      userId: req.user.id,
      category: mlResponse.data.category,
      priority: mlResponse.data.priority,
      assignedDepartment: mlResponse.data.assignedDepartment,
      status: 'open',
      confidence: mlResponse.data.confidence
    });
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});
```

### Scheduled Burnout Analysis

```javascript
// backend/jobs/burnoutCheck.js
const cron = require('node-cron');
const axios = require('axios');
const User = require('../models/User');

// Run every day at 9 AM
cron.schedule('0 9 * * *', async () => {
  console.log('Running daily burnout analysis...');
  
  const users = await User.find({ status: 'active' });
  
  for (const user of users) {
    const workload = await calculateUserWorkload(user._id);
    
    const analysis = await axios.post(`${process.env.ML_SERVICE_URL}/analyze/burnout`, workload);
    
    if (analysis.data.burnoutRisk === 'high') {
      // Send notification to admin and user
      await sendBurnoutAlert(user, analysis.data);
    }
  }
});

async function calculateUserWorkload(userId) {
  const tasks = await Task.find({ assignedTo: userId });
  const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
  
  return {
    userId: userId.toString(),
    tasksAssigned: tasks.length,
    tasksCompleted: tasks.filter(t => t.status === 'done').length,
    avgWorkHoursPerDay: await calculateAvgWorkHours(userId),
    overtimeHours: await calculateOvertimeHours(userId),
    missedDeadlines: tasks.filter(t => t.dueDate < new Date() && t.status !== 'done').length,
    lastVacationDays: await calculateDaysSinceVacation(userId)
  };
}
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
TRAINING_INTERVAL=86400
MIN_CONFIDENCE_THRESHOLD=0.75
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
```

## Troubleshooting

### ML Service Issues

**Model not found error:**
```python
# Initialize models if not exist
import os
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib

MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Create dummy model if training data unavailable
if not os.path.exists(f"{MODEL_PATH}/ticket_classifier.pkl"):
    print("Initializing default models...")
    vectorizer = TfidfVectorizer(max_features=1000)
    classifier = MultinomialNB()
    
    # Minimal training data
    sample_texts = ["technical issue", "billing problem", "account question"]
    sample_labels = ["technical", "billing", "account"]
    
    X = vectorizer.fit_transform(sample_texts)
    classifier.fit(X, sample_labels)
    
    joblib.dump(vectorizer, f"{MODEL_PATH}/ticket_vectorizer.pkl")
    joblib.dump(classifier, f"{MODEL_PATH}/ticket_classifier.pkl")
```

**MongoDB connection issues:**
```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    // Retry connection after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

module.exports = connectDB;
```

**CORS errors:**
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Performance Optimization

**Caching frequent queries:**
```javascript
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 }); // 10 minutes

app.get('/api/analytics/dashboard', authenticate, async (req, res) => {
  const cacheKey = `dashboard_${req.user.id}`;
  const cached = cache.get(cacheKey);
  
  if (cached) {
    return res.json({ success: true, data: cached, cached: true });
  }
  
  const analytics = await generateDashboardData(req.user.id);
  cache.set(cacheKey, analytics);
  
  res.json({ success: true, data: analytics });
});
```

**Batch AI predictions:**
```python
# Process multiple tickets at once
@app.post("/classify/tickets/batch")
async def classify_tickets_batch(tickets: List[TicketRequest]):
    results = []
    texts = [f"{t.title} {t.description}" for t in tickets]
    
    vectorizer = joblib.load(f"{MODEL_PATH}/ticket_vectorizer.pkl")
    classifier = joblib.load(f"{MODEL_PATH}/ticket_classifier.pkl")
    
    features = vectorizer.transform(texts)
    categories = classifier.predict(features)
    confidences = classifier.predict_proba(features).max(axis=1)
    
    for i, ticket in enumerate(tickets):
        results.append({
            "ticketId": ticket.userId,
            "category": categories[i],
            "priority": determine_priority(texts[i]),
            "confidence": float(confidences[i]),
            "assignedDepartment": map_category_to_department(categories[i])
        })
    
    return results
```

This skill provides comprehensive
