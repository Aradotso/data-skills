---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "build an enterprise user management system"
  - "implement AI analytics for user management"
  - "create a task management system with AI insights"
  - "set up user dashboard with kanban board"
  - "integrate ML service for risk detection"
  - "add AI-powered ticket classification"
  - "build user management with burnout detection"
  - "implement predictive analytics for project management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI/ML capabilities including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Role-based access control, authentication via JWT, user CRUD operations
- **Task Management**: Kanban board (To Do → In Progress → Done), time tracking, task assignment
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Separate admin and user dashboards with real-time insights
- **Audit Logging**: Track user activities and system events

The system consists of three main components:
1. **Frontend** (React.js) - User interface and dashboards
2. **Backend** (Node.js/Express) - API server and business logic
3. **ML Service** (FastAPI + scikit-learn) - AI/ML models and predictions

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB
- Git

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `.env` file:

```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start the backend:

```bash
npm start
# Or for development with auto-reload
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start the ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start the frontend:

```bash
npm start
```

Access the application at `http://localhost:3000`

## Key API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
```

### User Management

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Get user by ID
GET /api/users/:id

// Update user
PUT /api/users/:id
{
  "username": "john.updated",
  "role": "admin"
}

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Get user tasks
GET /api/tasks/user/:userId
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Login issue",
  "description": "Cannot login with credentials",
  "priority": "high",
  "category": "technical"
}

// Get ticket with AI classification
GET /api/tickets/:id/classify
// Returns AI-suggested category and priority
```

### AI Analytics Endpoints

```javascript
// Risk prediction
POST /api/ml/predict-risk
{
  "userId": "user_id",
  "recentActivities": [...],
  "taskLoad": 15
}

// Anomaly detection
POST /api/ml/detect-anomaly
{
  "userId": "user_id",
  "loginTime": "2026-04-15T03:30:00Z",
  "ipAddress": "192.168.1.1",
  "actionType": "data_export"
}

// Burnout analysis
POST /api/ml/burnout-analysis
{
  "userId": "user_id",
  "tasksCompleted": 45,
  "averageWorkHours": 12,
  "overtimeHours": 20
}

// Project delay prediction
POST /api/ml/predict-delay
{
  "projectId": "proj_123",
  "completionPercentage": 45,
  "daysRemaining": 10,
  "teamSize": 5
}
```

## Frontend Code Examples

### Using Authentication Context

```javascript
// src/contexts/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      console.error('Auth error:', error);
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/user/${userId}`);
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const onDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const onDrop = async (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    await updateTaskStatus(taskId, status);
  };

  const onDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="kanban-board">
      <div className="kanban-column" onDrop={(e) => onDrop(e, 'todo')} onDragOver={onDragOver}>
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <div key={task._id} className="task-card" draggable onDragStart={(e) => onDragStart(e, task._id)}>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority-${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>
      
      <div className="kanban-column" onDrop={(e) => onDrop(e, 'in-progress')} onDragOver={onDragOver}>
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <div key={task._id} className="task-card" draggable onDragStart={(e) => onDragStart(e, task._id)}>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority-${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>
      
      <div className="kanban-column" onDrop={(e) => onDrop(e, 'done')} onDragOver={onDragOver}>
        <h3>Done</h3>
        {tasks.done.map(task => (
          <div key={task._id} className="task-card" draggable onDragStart={(e) => onDragStart(e, task._id)}>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority-${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Line, Bar } from 'react-chartjs-2';

const AIAnalytics = ({ userId }) => {
  const [riskScore, setRiskScore] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [anomalies, setAnomalies] = useState([]);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Get risk prediction
      const riskResponse = await axios.post(`${process.env.REACT_APP_ML_URL}/predict-risk`, {
        userId: userId,
        recentActivities: await getRecentActivities(userId),
        taskLoad: await getTaskLoad(userId)
      });
      setRiskScore(riskResponse.data.riskScore);

      // Get burnout analysis
      const burnoutResponse = await axios.post(`${process.env.REACT_APP_ML_URL}/burnout-analysis`, {
        userId: userId,
        tasksCompleted: riskResponse.data.tasksCompleted,
        averageWorkHours: riskResponse.data.avgWorkHours,
        overtimeHours: riskResponse.data.overtimeHours
      });
      setBurnoutData(burnoutResponse.data);

      // Get anomalies
      const anomalyResponse = await axios.get(`${process.env.REACT_APP_API_URL}/analytics/anomalies/${userId}`);
      setAnomalies(anomalyResponse.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const getRecentActivities = async (userId) => {
    const response = await axios.get(`${process.env.REACT_APP_API_URL}/activities/user/${userId}?limit=50`);
    return response.data;
  };

  const getTaskLoad = async (userId) => {
    const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/user/${userId}/count`);
    return response.data.count;
  };

  return (
    <div className="ai-analytics">
      <h2>AI Analytics Dashboard</h2>
      
      <div className="analytics-grid">
        <div className="analytics-card">
          <h3>Risk Score</h3>
          <div className={`risk-indicator ${riskScore > 0.7 ? 'high' : riskScore > 0.4 ? 'medium' : 'low'}`}>
            {riskScore ? (riskScore * 100).toFixed(1) : '—'}%
          </div>
          <p>{riskScore > 0.7 ? 'High Risk' : riskScore > 0.4 ? 'Medium Risk' : 'Low Risk'}</p>
        </div>

        <div className="analytics-card">
          <h3>Burnout Risk</h3>
          {burnoutData && (
            <>
              <div className={`burnout-indicator ${burnoutData.burnoutRisk > 0.6 ? 'high' : 'normal'}`}>
                {(burnoutData.burnoutRisk * 100).toFixed(1)}%
              </div>
              <p>Avg Hours: {burnoutData.avgWorkHours}h/day</p>
              <p>Overtime: {burnoutData.overtimeHours}h/week</p>
            </>
          )}
        </div>

        <div className="analytics-card">
          <h3>Recent Anomalies</h3>
          <ul className="anomaly-list">
            {anomalies.slice(0, 5).map((anomaly, idx) => (
              <li key={idx} className={`anomaly-${anomaly.severity}`}>
                {anomaly.description}
                <span className="anomaly-time">{new Date(anomaly.timestamp).toLocaleDateString()}</span>
              </li>
            ))}
          </ul>
        </div>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Code Examples

### User Model (MongoDB/Mongoose)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: {
    type: String,
    default: ''
  },
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    default: ''
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  dueDate: {
    type: Date
  },
  timeTracked: {
    type: Number,
    default: 0 // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId).select('-password');

    if (!user || !user.isActive) {
      return res.status(401).json({ error: 'Invalid authentication token' });
    }

    req.user = user;
    req.userId = user._id;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Authentication failed' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.userId,
      priority,
      dueDate,
      status: 'todo'
    });

    await task.save();
    await task.populate('assignedTo createdBy', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const { userId } = req.params;
    const tasks = await Task.find({ assignedTo: userId })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;
    
    const task = await Task.findById(id);
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.status = status;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const { id } = req.params;
    const { seconds } = req.body;
    
    const task = await Task.findById(id);
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.timeTracked += seconds;
    await task.save();
    
    res.json({ timeTracked: task.timeTracked });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

## ML Service Code Examples

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class RiskPredictionRequest(BaseModel):
    userId: str
    recentActivities: List[dict]
    taskLoad: int

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    ipAddress: str
    actionType: str
    dataVolume: Optional[int] = 0

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    tasksCompleted: int
    averageWorkHours: float
    overtimeHours: float
    missedDeadlines: Optional[int] = 0

class ProjectDelayRequest(BaseModel):
    projectId: str
    completionPercentage: float
    daysRemaining: int
    teamSize: int
    taskBacklog: int

@app.get("/")
def read_root():
    return {"message": "Enterprise User Management ML Service", "version": "1.0.0"}

@app.post("/predict-risk")
def predict_risk(request: RiskPredictionRequest):
    try:
        # Feature engineering
        features = extract_risk_features(request)
        
        # Simple risk calculation (replace with trained model)
        base_risk = min(request.taskLoad / 20.0, 1.0)
        activity_risk = calculate_activity_risk(request.recentActivities)
        
        risk_score = (base_risk * 0.6) + (activity_risk * 0.4)
        
        return {
            "userId": request.userId,
            "riskScore": round(risk_score, 3),
            "riskLevel": "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low",
            "factors": {
                "taskLoad": request.taskLoad,
                "activityRisk": round(activity_risk, 3)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        features = extract_anomaly_features(request)
        
        # Simple anomaly detection (replace with Isolation Forest)
        is_anomaly = False
        anomaly_score = 0.0
        reasons = []
        
        # Check for unusual login time
        hour = int(request.loginTime.split('T')[1].split(':')[0])
        if hour < 6 or hour > 22:
            is_anomaly = True
            anomaly_score += 0.4
            reasons.append("Unusual login time")
        
        # Check for suspicious action
        if request.actionType in ['data_export', 'bulk_delete', 'permission_change']:
            is_anomaly = True
            anomaly_score += 0.3
            reasons.append(f"Sensitive action: {request.actionType}")
        
        # Check data volume
        if request.dataVolume > 10000:
            is_anomaly = True
            anomaly_score += 0.3
            reasons.append("Large data volume")
        
        return {
            "userId": request.userId,
            "isAnomaly": is_anomaly,
            "anomalyScore": min(anomaly_score, 1.0),
            "reasons": reasons,
            "severity": "high" if anomaly_score > 0.7 else "medium" if anomaly_score > 0.4 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/burnout-analysis")
def burnout_analysis(request: BurnoutAnalysisRequest):
    try:
        # Calculate burnout risk
        workload_factor = min(request.averageWorkHours / 12.0, 1.0)
        overtime_factor = min(request.overtimeHours / 20.0, 1.0)
        deadline_factor = min(request.missedDeadlines / 5.0, 1.0)
        
        burnout_risk = (workload_factor * 0.4) + (overtime_factor * 0.4) + (deadline_factor * 0.2)
        
        recommendations = []
        if burnout_risk > 0.6:
            recommendations.append("Reduce workload immediately")
            recommendations.append("Schedule time off")
            recommendations.append("Redistribute tasks to team members")
        elif burnout_risk > 0.4:
            recommendations.append("Monitor work hours closely")
            recommendations.append("Consider workload adjustment")
        
        return {
            "userId": request.userId,
            "burnoutRisk": round(burnout_risk, 3),
            "riskLevel": "critical" if burnout_risk > 0.7 else "high" if burnout_risk > 0.5 else "moderate",
            "factors": {
                "avgWorkHours": request.averageWorkHours,
                "overtimeHours": request.overtimeHours,
                "missedDeadlines": request.missedDeadlines
            },
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-delay")
def predict_project_delay(request: ProjectDelayRequest):
    try:
        # Calculate delay probability
        pace = request.completionPercentage / max(1, 100 - request.daysRemaining)
        required_pace = 1.0 / max(1, request.daysRemaining)
        
        team_efficiency = min(request.teamSize / 5.0, 1.0)
        backlog_pressure = min(request.taskBacklog / 20.0, 1.0)
        
        delay_probability = 0.0
        
        if pace < required_pace:
            delay_probability += 0.5
        
        delay_probability += backlog_pressure * 0.3
        delay_probability -= team_efficiency * 0.2
        
        delay_probability = max(0, min(delay_probability, 1.0))
        
        estimated_delay_days = 0
        if delay_probability > 0.5:
            estimated_delay_days = int((required_pace - pace) * request.daysRemaining * 10)
        
        return {
            "projectId": request.projectId,
            "delayProbability": round(delay_probability, 3),
            "estimatedDelayDays": estimated_delay_days,
            "riskLevel": "high" if delay_probability > 0.7 else "medium" if delay_probability > 0.4 else "low",
            "factors": {
                "currentPace": round(pace, 3),
                "requiredPace": round(required_pace, 3),
                "teamEfficiency": round(team_efficiency, 3),
                "backlogPressure": round(backlog_pressure, 3)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def extract_risk_features(request: RiskPredictionRequest):
    return {
        "task_load": request.taskLoad,
        "activity_count": len(request.recentActivities)
    }

def calculate_activity_risk(activities: List[dict]):
    if not activities:
        return 0.0
    
    risky_types = ['data_export', 'bulk_delete', 'permission_change']
    risky_count = sum(1 for act in activities if act.get('type') in risky_types)
    
    return min(risky_count / len(activities), 1.0)

def extract_anomaly_features(request: AnomalyDetectionRequest):
    return {
        "login_time": request.loginTime,
        "ip_address": request.ipAddress,
        "action_type": request.actionType,
        "data_volume": request.dataVolume
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Ticket Classification Model

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import joblib
import os

class TicketClassifier:
    def __init__(self, model_path='
