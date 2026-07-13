---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system with AI"
  - "integrate AI analytics for user management"
  - "configure task tracking and ticket management system"
  - "implement risk detection and anomaly monitoring"
  - "deploy user management platform with ML service"
  - "create admin dashboard with AI insights"
  - "build kanban board with time tracking"
  - "add AI-powered ticket classification"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning capabilities. It provides:

- **User Management**: Role-based access control, authentication, user CRUD operations
- **Task Management**: Kanban boards, time tracking, assignment workflows
- **Support Tickets**: Smart ticket routing and classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay forecasting
- **Admin Dashboard**: Organization-wide analytics and audit logs

The system consists of three main components:
1. **Frontend** (React.js) - User interface and dashboards
2. **Backend** (Node.js/Express) - REST API and business logic
3. **ML Service** (FastAPI + scikit-learn) - AI/ML predictions and insights

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB 4.4+

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
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
DATA_PATH=./data
LOG_LEVEL=INFO
EOF

uvicorn main:app --reload --port 8000
```

ML service runs at `http://localhost:8000`

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

Frontend runs at `http://localhost:3000`

## Backend API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "SecurePass123",
  "role": "user"
}

// Login
POST /api/auth/login
Content-Type: application/json

{
  "email": "john@company.com",
  "password": "SecurePass123"
}

// Response includes JWT token
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": { "id": "...", "name": "John Doe", "role": "user" }
}
```

### User Management Endpoints

```javascript
// Get all users (Admin only)
GET /api/users
Authorization: Bearer <token>

// Get user by ID
GET /api/users/:id
Authorization: Bearer <token>

// Update user
PUT /api/users/:id
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "John Updated",
  "role": "manager"
}

// Delete user (Admin only)
DELETE /api/users/:id
Authorization: Bearer <token>
```

### Task Management Endpoints

```javascript
// Create task
POST /api/tasks
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "Implement authentication",
  "description": "Add JWT-based auth system",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Get tasks
GET /api/tasks
Authorization: Bearer <token>

// Update task status
PUT /api/tasks/:id
Authorization: Bearer <token>
Content-Type: application/json

{
  "status": "in_progress"
}

// Track time on task
POST /api/tasks/:id/time
Authorization: Bearer <token>
Content-Type: application/json

{
  "timeSpent": 3600 // seconds
}
```

### Ticket Management Endpoints

```javascript
// Create support ticket
POST /api/tickets
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when accessing /dashboard",
  "priority": "high",
  "category": "technical"
}

// Get tickets
GET /api/tickets
Authorization: Bearer <token>

// Update ticket
PUT /api/tickets/:id
Authorization: Bearer <token>
Content-Type: application/json

{
  "status": "in_progress",
  "assignedTo": "admin_id"
}
```

## ML Service API Reference

### Risk Prediction

```python
# Risk prediction endpoint
POST /api/ml/predict-risk
Content-Type: application/json

{
  "userId": "user123",
  "taskCompletionRate": 0.75,
  "avgResponseTime": 1800,
  "issuesRaised": 3,
  "workHours": 50
}

# Response
{
  "riskScore": 0.68,
  "riskLevel": "medium",
  "factors": ["high_work_hours", "low_completion_rate"]
}
```

### Anomaly Detection

```python
# Anomaly detection
POST /api/ml/detect-anomaly
Content-Type: application/json

{
  "userId": "user123",
  "loginTime": "2026-04-15T03:00:00Z",
  "ipAddress": "192.168.1.100",
  "actions": ["delete_user", "modify_permissions"]
}

# Response
{
  "isAnomaly": true,
  "anomalyScore": 0.85,
  "reasons": ["unusual_login_time", "suspicious_actions"]
}
```

### Burnout Analysis

```python
# Burnout prediction
POST /api/ml/burnout-analysis
Content-Type: application/json

{
  "userId": "user123",
  "weeklyHours": [45, 52, 58, 62, 65],
  "tasksCompleted": [8, 7, 6, 5, 4],
  "stressIndicators": {
    "missedDeadlines": 3,
    "overtimeHours": 25
  }
}

# Response
{
  "burnoutRisk": "high",
  "burnoutScore": 0.78,
  "recommendations": [
    "Reduce workload by 20%",
    "Schedule time off",
    "Redistribute tasks"
  ]
}
```

### Ticket Classification

```python
# Auto-classify support ticket
POST /api/ml/classify-ticket
Content-Type: application/json

{
  "title": "Cannot login to system",
  "description": "Getting authentication error when trying to access the platform"
}

# Response
{
  "category": "authentication",
  "priority": "high",
  "suggestedAssignee": "tech_support_team",
  "confidence": 0.92
}
```

### Project Delay Prediction

```python
# Predict project delays
POST /api/ml/predict-delay
Content-Type: application/json

{
  "projectId": "proj123",
  "totalTasks": 50,
  "completedTasks": 20,
  "daysElapsed": 30,
  "daysRemaining": 20,
  "teamSize": 5,
  "avgVelocity": 0.67
}

# Response
{
  "delayPredicted": true,
  "estimatedDelay": 15,
  "confidenceScore": 0.83,
  "suggestions": [
    "Add 2 more developers",
    "Reduce scope by 10 tasks"
  ]
}
```

## Frontend Integration Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data.user);
    } catch (err) {
      logout();
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    setToken(res.data.token);
    localStorage.setItem('token', res.data.token);
    setUser(res.data.user);
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
      const grouped = {
        todo: res.data.filter(t => t.status === 'todo'),
        in_progress: res.data.filter(t => t.status === 'in_progress'),
        done: res.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (err) {
      console.error('Error fetching tasks:', err);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (err) {
      console.error('Error updating task:', err);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status}
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in_progress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);

  useEffect(() => {
    fetchAnalytics();
    fetchRiskUsers();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/analytics`);
      setAnalytics(res.data);
    } catch (err) {
      console.error('Error fetching analytics:', err);
    }
  };

  const fetchRiskUsers = async () => {
    try {
      const usersRes = await axios.get(`${process.env.REACT_APP_API_URL}/api/users`);
      const users = usersRes.data;
      
      const riskAnalysis = await Promise.all(
        users.map(async (user) => {
          const mlRes = await axios.post(
            `${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`,
            {
              userId: user._id,
              taskCompletionRate: user.stats?.completionRate || 0,
              avgResponseTime: user.stats?.avgResponseTime || 0,
              issuesRaised: user.stats?.issuesRaised || 0,
              workHours: user.stats?.workHours || 40
            }
          );
          return { ...user, risk: mlRes.data };
        })
      );
      
      setRiskUsers(riskAnalysis.filter(u => u.risk.riskLevel === 'high'));
    } catch (err) {
      console.error('Error analyzing users:', err);
    }
  };

  return (
    <div className="admin-dashboard">
      <h2>Admin Dashboard</h2>
      
      {analytics && (
        <div className="analytics-summary">
          <div className="stat-card">
            <h3>Total Users</h3>
            <p>{analytics.totalUsers}</p>
          </div>
          <div className="stat-card">
            <h3>Active Tasks</h3>
            <p>{analytics.activeTasks}</p>
          </div>
          <div className="stat-card">
            <h3>Open Tickets</h3>
            <p>{analytics.openTickets}</p>
          </div>
        </div>
      )}

      <div className="risk-alerts">
        <h3>High Risk Users</h3>
        {riskUsers.map(user => (
          <div key={user._id} className="risk-card">
            <h4>{user.name}</h4>
            <p>Risk Score: {(user.risk.riskScore * 100).toFixed(0)}%</p>
            <p>Factors: {user.risk.factors.join(', ')}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Backend Implementation Examples

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Get all users
exports.getUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (err) {
    res.status(500).json({ message: 'Server error', error: err.message });
  }
};

// Get user by ID
exports.getUserById = async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json(user);
  } catch (err) {
    res.status(500).json({ message: 'Server error', error: err.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { name, email, role } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { name, email, role, updatedAt: Date.now() },
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json(user);
  } catch (err) {
    res.status(400).json({ message: 'Update failed', error: err.message });
  }
};

// Delete user
exports.deleteUser = async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json({ message: 'User deleted successfully' });
  } catch (err) {
    res.status(500).json({ message: 'Server error', error: err.message });
  }
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Create task
exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (err) {
    res.status(400).json({ message: 'Task creation failed', error: err.message });
  }
};

// Get tasks with filters
exports.getTasks = async (req, res) => {
  try {
    const { status, assignedTo, priority } = req.query;
    const filter = {};
    
    if (status) filter.status = status;
    if (assignedTo) filter.assignedTo = assignedTo;
    if (priority) filter.priority = priority;
    
    // Non-admin users only see their tasks
    if (req.user.role !== 'admin') {
      filter.$or = [
        { assignedTo: req.user.id },
        { createdBy: req.user.id }
      ];
    }
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (err) {
    res.status(500).json({ message: 'Server error', error: err.message });
  }
};

// Update task
exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { ...req.body, updatedAt: Date.now() },
      { new: true, runValidators: true }
    );
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (err) {
    res.status(400).json({ message: 'Update failed', error: err.message });
  }
};

// Track time
exports.trackTime = async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeTracking.push({
      userId: req.user.id,
      timeSpent,
      date: new Date()
    });
    
    await task.save();
    res.json(task);
  } catch (err) {
    res.status(400).json({ message: 'Time tracking failed', error: err.message });
  }
};
```

### Auth Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized to access this route` 
      });
    }
    next();
  };
};
```

## ML Service Implementation

### Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import numpy as np
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models'):
        self.model_path = model_path
        self.scaler = StandardScaler()
        self.model = None
        self.load_or_train()
    
    def load_or_train(self):
        model_file = os.path.join(self.model_path, 'risk_model.pkl')
        scaler_file = os.path.join(self.model_path, 'risk_scaler.pkl')
        
        if os.path.exists(model_file) and os.path.exists(scaler_file):
            self.model = joblib.load(model_file)
            self.scaler = joblib.load(scaler_file)
        else:
            self.train_initial_model()
    
    def train_initial_model(self):
        # Initialize with synthetic data for cold start
        self.model = RandomForestClassifier(n_estimators=100, random_state=42)
        # Train on initial dataset would happen here
        os.makedirs(self.model_path, exist_ok=True)
        joblib.dump(self.model, os.path.join(self.model_path, 'risk_model.pkl'))
        joblib.dump(self.scaler, os.path.join(self.model_path, 'risk_scaler.pkl'))
    
    def prepare_features(self, data):
        features = [
            data.get('taskCompletionRate', 0),
            data.get('avgResponseTime', 0) / 3600,  # Convert to hours
            data.get('issuesRaised', 0),
            data.get('workHours', 40) / 40,  # Normalize by standard work week
        ]
        return np.array(features).reshape(1, -1)
    
    def predict(self, data):
        features = self.prepare_features(data)
        features_scaled = self.scaler.fit_transform(features)
        
        # For demonstration, use heuristic scoring
        completion_rate = data.get('taskCompletionRate', 0)
        work_hours = data.get('workHours', 40)
        issues = data.get('issuesRaised', 0)
        
        risk_score = 0
        factors = []
        
        if completion_rate < 0.6:
            risk_score += 0.3
            factors.append('low_completion_rate')
        
        if work_hours > 50:
            risk_score += 0.25
            factors.append('high_work_hours')
        
        if issues > 5:
            risk_score += 0.2
            factors.append('many_issues')
        
        if data.get('avgResponseTime', 0) > 7200:
            risk_score += 0.25
            factors.append('slow_response')
        
        risk_level = 'low'
        if risk_score > 0.7:
            risk_level = 'high'
        elif risk_score > 0.4:
            risk_level = 'medium'
        
        return {
            'riskScore': min(risk_score, 1.0),
            'riskLevel': risk_level,
            'factors': factors
        }
```

### Burnout Analysis

```python
# ml-service/models/burnout_analyzer.py
import numpy as np
from datetime import datetime, timedelta

class BurnoutAnalyzer:
    def __init__(self):
        self.threshold_hours = 50
        self.threshold_overtime = 15
    
    def analyze(self, data):
        weekly_hours = data.get('weeklyHours', [40] * 5)
        tasks_completed = data.get('tasksCompleted', [5] * 5)
        stress_indicators = data.get('stressIndicators', {})
        
        # Calculate trends
        avg_hours = np.mean(weekly_hours)
        hours_trend = self._calculate_trend(weekly_hours)
        tasks_trend = self._calculate_trend(tasks_completed)
        
        burnout_score = 0
        recommendations = []
        
        # High average hours
        if avg_hours > self.threshold_hours:
            burnout_score += 0.3
            recommendations.append('Reduce workload by 20%')
        
        # Increasing hours trend
        if hours_trend > 0.1:
            burnout_score += 0.2
            recommendations.append('Monitor work-life balance')
        
        # Decreasing productivity
        if tasks_trend < -0.1:
            burnout_score += 0.25
            recommendations.append('Review task allocation')
        
        # Missed deadlines
        missed_deadlines = stress_indicators.get('missedDeadlines', 0)
        if missed_deadlines > 2:
            burnout_score += 0.15
            recommendations.append('Extend project timelines')
        
        # Overtime hours
        overtime = stress_indicators.get('overtimeHours', 0)
        if overtime > self.threshold_overtime:
            burnout_score += 0.1
            recommendations.append('Schedule time off')
        
        risk_level = 'low'
        if burnout_score > 0.7:
            risk_level = 'high'
            recommendations.append('Immediate intervention required')
        elif burnout_score > 0.4:
            risk_level = 'medium'
        
        return {
            'burnoutRisk': risk_level,
            'burnoutScore': min(burnout_score, 1.0),
            'recommendations': recommendations
        }
    
    def _calculate_trend(self, values):
        """Calculate linear trend coefficient"""
        if len(values) < 2:
            return 0
        x = np.arange(len(values))
        z = np.polyfit(x, values, 1)
        return z[0]
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
from models.risk_predictor import RiskPredictor
from models.burnout_analyzer import BurnoutAnalyzer
import logging

app = FastAPI(title="Enterprise User Management ML Service")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize models
risk_predictor = RiskPredictor()
burnout_analyzer = BurnoutAnalyzer()

# Pydantic models
class RiskPredictionRequest(BaseModel):
    userId: str
    taskCompletionRate: float
    avgResponseTime: int
    issuesRaised: int
    workHours: int

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    weeklyHours: List[int]
    tasksCompleted: List[int]
    stressIndicators: Dict[str, int]

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class ProjectDelayRequest(BaseModel):
    projectId: str
    totalTasks: int
    completedTasks: int
    daysElapsed: int
    daysRemaining: int
    teamSize: int
    avgVelocity: float

@app.get("/")
def root():
    return {"message": "Enterprise User Management ML Service", "status": "active"}

@app.post("/api/ml/predict-risk")
def predict_risk(request: RiskPredictionRequest):
    try:
        result = risk_predictor.predict(request.dict())
        return result
    except Exception as e:
        logging.error(f"Risk prediction error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        result = burnout_analyzer.analyze(request.dict())
        return result
    except Exception as e:
        logging.error(f"Burnout analysis error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
def classify_ticket(request: TicketClassificationRequest):
    try:
        # Simple keyword-based classification
        text = (request.title + " " + request.description).lower()
        
        categories = {
            'authentication': ['login', 'password', 'auth', 'access denied', 'credentials'],
            'technical': ['error', 'bug', 'crash', 'not working', 'broken'],
            'feature': ['request', 'enhancement', 'improvement', 'add'],
            'data': ['database', 'data', 'export', 'import', 'sync']
        }
        
        category = 'general'
        max_matches = 0
        
        for cat, keywords in categories.items():
            matches = sum(1 for kw in keywords if kw in text)
            if matches > max_matches:
                max_matches = matches
                category = cat
        
        priority = 'high' if any(word in text for word in ['urgent', 'critical', 'cannot', 'error']) else 'medium
