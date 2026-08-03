---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, task management, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement user task tracking with AI insights"
  - "create admin dashboard with anomaly detection"
  - "build kanban board with burnout analysis"
  - "integrate AI ticket classification system"
  - "configure user management with risk prediction"
  - "deploy enterprise system with ML service"
  - "add AI-powered support ticket routing"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, user CRUD operations, authentication via JWT
- **Task Management**: Kanban board (To Do → In Progress → Done), time tracking, task assignment
- **Support Tickets**: Ticket creation, tracking, and AI-based classification/routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Features**: Organization analytics dashboard, audit logs, user monitoring
- **ML Service**: FastAPI-based microservice for AI/ML predictions using scikit-learn and River

## Installation

### Prerequisites

```bash
# Required software
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**Backend (.env)**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env)**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

**ML Service (.env)**
```bash
# ml-service/.env
MODEL_PATH=./models
LOG_LEVEL=INFO
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
```

## Running the System

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs on http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload
# Runs on http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Runs on http://localhost:3000
```

### Production Build

```bash
# Frontend production build
cd frontend
npm run build

# Backend production
cd backend
NODE_ENV=production npm start

# ML Service production
cd ml-service
uvicorn main:app --host 0.0.0.0 --port 8000
```

## Key Backend API Endpoints

### Authentication

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

```javascript
// GET /api/users - List all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority,
      dueDate: taskData.dueDate,
      status: 'todo'
    })
  });
  return response.json();
};

// GET /api/tasks - Get user tasks
const getUserTasks = async (token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus }) // 'todo', 'in-progress', 'done'
  });
  return response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets (admin) or user tickets
const getTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

## ML Service API

### AI-Powered Analytics

```python
# Example ML service endpoints (FastAPI)
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np

app = FastAPI()

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    suspicious_activities: int
    access_anomalies: int
    task_completion_rate: float

@app.post("/api/ml/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior patterns"""
    features = np.array([[
        request.failed_logins,
        request.suspicious_activities,
        request.access_anomalies,
        request.task_completion_rate
    ]])
    
    # Load pre-trained model
    model = joblib.load('models/risk_model.pkl')
    risk_score = model.predict_proba(features)[0][1]
    
    return {
        "user_id": request.user_id,
        "risk_score": float(risk_score),
        "risk_level": "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
    }

class BurnoutRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_working_hours: float
    missed_deadlines: int

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutRequest):
    """Detect employee burnout risk"""
    workload_ratio = request.tasks_assigned / max(request.tasks_completed, 1)
    burnout_score = (
        workload_ratio * 0.3 +
        (request.avg_working_hours / 8.0) * 0.3 +
        (request.missed_deadlines / 10.0) * 0.4
    )
    
    return {
        "user_id": request.user_id,
        "burnout_score": min(burnout_score, 1.0),
        "recommendation": "Reduce workload" if burnout_score > 0.7 else "Normal"
    }

class TicketClassificationRequest(BaseModel):
    ticket_text: str

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket category"""
    # Load vectorizer and classifier
    vectorizer = joblib.load('models/ticket_vectorizer.pkl')
    classifier = joblib.load('models/ticket_classifier.pkl')
    
    text_vector = vectorizer.transform([request.ticket_text])
    category = classifier.predict(text_vector)[0]
    confidence = max(classifier.predict_proba(text_vector)[0])
    
    return {
        "category": category,
        "confidence": float(confidence),
        "suggested_department": get_department(category)
    }

def get_department(category):
    mapping = {
        "technical": "IT Support",
        "hr": "Human Resources",
        "finance": "Finance",
        "general": "General Support"
    }
    return mapping.get(category, "General Support")
```

### Calling ML Service from Frontend

```javascript
// Call risk prediction
const getRiskPrediction = async (userId, behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      failed_logins: behaviorData.failedLogins,
      suspicious_activities: behaviorData.suspiciousActivities,
      access_anomalies: behaviorData.accessAnomalies,
      task_completion_rate: behaviorData.taskCompletionRate
    })
  });
  return response.json();
};

// Call burnout detection
const checkBurnout = async (userId, workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      tasks_assigned: workloadData.tasksAssigned,
      tasks_completed: workloadData.tasksCompleted,
      avg_working_hours: workloadData.avgWorkingHours,
      missed_deadlines: workloadData.missedDeadlines
    })
  });
  return response.json();
};

// Auto-classify ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ticket_text: ticketText })
  });
  return response.json();
};
```

## Common React Component Patterns

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch('http://localhost:5000/api/tasks', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Organize tasks by status
    const organized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(organized);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks(); // Refresh
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={moveTask} targetStatus="todo" />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={moveTask} targetStatus="in-progress" />
      <Column title="Done" tasks={tasks.done} onMove={moveTask} targetStatus="done" />
    </div>
  );
};

const Column = ({ title, tasks, onMove, targetStatus }) => (
  <div className="kanban-column">
    <h3>{title}</h3>
    {tasks.map(task => (
      <TaskCard key={task._id} task={task} onMove={onMove} />
    ))}
  </div>
);
```

### Admin Dashboard with AI Insights

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    // Fetch organization analytics
    const statsResponse = await fetch('http://localhost:5000/api/admin/stats', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const stats = await statsResponse.json();
    setAnalytics(stats);

    // Fetch high-risk users from ML service
    const usersResponse = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const users = await usersResponse.json();

    // Get risk predictions for each user
    const riskPromises = users.map(async (user) => {
      const riskData = await fetch('http://localhost:8000/api/ml/risk-prediction', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          user_id: user._id,
          failed_logins: user.failedLogins || 0,
          suspicious_activities: user.suspiciousActivities || 0,
          access_anomalies: user.accessAnomalies || 0,
          task_completion_rate: user.taskCompletionRate || 0.8
        })
      });
      const risk = await riskData.json();
      return { ...user, risk };
    });

    const usersWithRisk = await Promise.all(riskPromises);
    setRiskUsers(usersWithRisk.filter(u => u.risk.risk_level === 'high'));
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <StatCard title="Total Users" value={analytics?.totalUsers} />
        <StatCard title="Active Tasks" value={analytics?.activeTasks} />
        <StatCard title="Pending Tickets" value={analytics?.pendingTickets} />
        <StatCard title="High Risk Users" value={riskUsers.length} />
      </div>

      <div className="risk-alerts">
        <h2>High Risk Users</h2>
        {riskUsers.map(user => (
          <div key={user._id} className="risk-alert">
            <span>{user.name}</span>
            <span>Risk Score: {(user.risk.risk_score * 100).toFixed(1)}%</span>
            <button onClick={() => reviewUser(user._id)}>Review</button>
          </div>
        ))}
      </div>
    </div>
  );
};
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const token = localStorage.getItem('token');

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const toggleTimer = () => {
    setIsRunning(!isRunning);
  };

  const saveTime = async () => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ timeSpent: seconds })
    });
    setSeconds(0);
    setIsRunning(false);
  };

  const formatTime = (secs) => {
    const hrs = Math.floor(secs / 3600);
    const mins = Math.floor((secs % 3600) / 60);
    const s = secs % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <button onClick={toggleTimer}>{isRunning ? 'Pause' : 'Start'}</button>
      <button onClick={saveTime} disabled={seconds === 0}>Save</button>
    </div>
  );
};
```

## MongoDB Schema Examples

### User Schema

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  failedLogins: { type: Number, default: 0 },
  suspiciousActivities: { type: Number, default: 0 },
  accessAnomalies: { type: Number, default: 0 },
  taskCompletionRate: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  lastLogin: { type: Date }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Schema

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: { type: Date },
  timeSpent: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Schema

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  category: { type: String },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: {
    category: String,
    confidence: Number,
    department: String
  },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Training ML Models

### Risk Prediction Model

```python
# ml-service/train_risk_model.py
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report
import joblib

def train_risk_model():
    # Load historical data from MongoDB
    # Features: failed_logins, suspicious_activities, access_anomalies, task_completion_rate
    # Target: is_risky (0 or 1)
    
    # Example synthetic data
    data = pd.DataFrame({
        'failed_logins': np.random.randint(0, 10, 1000),
        'suspicious_activities': np.random.randint(0, 5, 1000),
        'access_anomalies': np.random.randint(0, 8, 1000),
        'task_completion_rate': np.random.uniform(0.3, 1.0, 1000),
        'is_risky': np.random.randint(0, 2, 1000)
    })
    
    X = data.drop('is_risky', axis=1)
    y = data['is_risky']
    
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
    
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)
    
    print(classification_report(y_test, model.predict(X_test)))
    
    # Save model
    joblib.dump(model, 'models/risk_model.pkl')
    print("Risk model saved to models/risk_model.pkl")

if __name__ == "__main__":
    train_risk_model()
```

### Ticket Classification Model

```python
# ml-service/train_ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import joblib

def train_ticket_classifier():
    # Training data
    tickets = [
        "Cannot access VPN",
        "Password reset needed",
        "Software installation request",
        "Salary inquiry",
        "Leave approval",
        "Invoice payment issue",
        "Budget allocation question",
        "General query about policy"
    ]
    
    categories = [
        "technical", "technical", "technical",
        "hr", "hr",
        "finance", "finance",
        "general"
    ]
    
    vectorizer = TfidfVectorizer(max_features=100)
    classifier = MultinomialNB()
    
    X = vectorizer.fit_transform(tickets)
    classifier.fit(X, categories)
    
    # Save models
    joblib.dump(vectorizer, 'models/ticket_vectorizer.pkl')
    joblib.dump(classifier, 'models/ticket_classifier.pkl')
    print("Ticket classifier saved")

if __name__ == "__main__":
    train_ticket_classifier()
```

## Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    res.status(401).json({ message: 'Not authorized, token failed' });
  }
};

const admin = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ message: 'Not authorized as admin' });
  }
};

module.exports = { protect, admin };
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB Connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Configuration

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: process.env.NODE_ENV === 'production' 
    ? 'https://enterprise-user-management-system-w.vercel.app'
    : 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Model Loading

```python
# ml-service/main.py
import os
from pathlib import Path

# Ensure models directory exists
MODEL_DIR = Path("models")
MODEL_DIR.mkdir(exist_ok=True)

# Check if models exist before loading
def load_model_safely(model_path):
    if not os.path.exists(model_path):
        raise FileNotFoundError(f"Model not found: {model_path}. Run training script first.")
    return joblib.load(model_path)
```

### JWT Token Expiration Handling

```javascript
// frontend/utils/api.js
const fetchWithAuth = async (url, options = {}) => {
  const token = localStorage.getItem('token');
  
  const response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`
    }
  });

  if (response.status === 401) {
    // Token expired, redirect to login
    localStorage.removeItem('token');
    window.location.href = '/login';
    throw new Error('Session expired');
  }

  return response;
};
```

## Deployment

### Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:latest
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise_user_mgmt
      - JWT_SECRET=${JWT_SECRET}
      - ML_SERVICE_URL=http://ml-service:8000
    depends_on:
      - mongodb

  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise_user_mgmt
    depends_on:
      - mongodb

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:5000
      - REACT_APP_ML_URL=http://localhost:8000
    depends_on:
      - backend
      - ml-service

volumes:
  mongo-data:
```

Run with:
```bash
docker-compose up -d
```
