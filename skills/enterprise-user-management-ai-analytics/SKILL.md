---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task management, ticket routing, and risk detection
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with AI insights"
  - "implement AI-powered ticket classification system"
  - "build task management with burnout detection"
  - "configure user authentication with JWT and role-based access"
  - "integrate AI analytics for risk prediction and anomaly detection"
  - "deploy user management system with ML service"
  - "set up Kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack application that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, authentication, and user administration
- **Task Management**: Kanban board, time tracking, and progress monitoring
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Dashboards**: Separate admin and user interfaces with real-time insights

## Architecture Overview

The system consists of three main components:
- **Frontend**: React.js application (port 3000)
- **Backend**: Node.js REST API with MongoDB (port 5000)
- **ML Service**: FastAPI-based AI/ML microservice (port 8000)

## Installation

### Prerequisites
```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB
mongod --version
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install all components
cd backend && npm install && cd ..
cd frontend && npm install && cd ..
cd ml-service && pip install -r requirements.txt && cd ..
```

### Environment Configuration

**Backend** (`.env` in `backend/`):
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend** (`.env` in `frontend/`):
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

**ML Service** (`.env` in `ml-service/`):
```bash
MODEL_PATH=./models
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
LOG_LEVEL=INFO
```

## Running the Application

### Start All Services

```bash
# Terminal 1 - MongoDB
mongod --dbpath /path/to/data/db

# Terminal 2 - Backend
cd backend
npm start

# Terminal 3 - ML Service
cd ml-service
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# Terminal 4 - Frontend
cd frontend
npm start
```

## Backend API Reference

### Authentication Endpoints

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
    })
  });
  return await response.json();
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
// GET /api/users - Get all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
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
  return await response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
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
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in-progress', 'done'
    })
  });
  return await response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status })
  });
  return await response.json();
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
  return await response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async (token, filters = {}) => {
  const queryParams = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${queryParams}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

## ML Service API Reference

### AI Ticket Classification

```javascript
// POST /classify-ticket - Classify and route ticket using AI
const classifyTicket = async (ticketData) => {
  const response = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority
    })
  });
  const result = await response.json();
  // Returns: { category, confidence, suggested_assignee }
  return result;
};
```

### Risk Detection

```javascript
// POST /predict-risk - Predict user risk level
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        login_frequency: 5,
        failed_login_attempts: 2,
        task_completion_rate: 0.85,
        avg_response_time: 120, // minutes
        tickets_raised: 3
      }
    })
  });
  const result = await response.json();
  // Returns: { risk_level: 'low'|'medium'|'high', risk_score: 0.0-1.0 }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /detect-anomaly - Detect anomalous behavior
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userActivity.userId,
      activity_data: {
        login_time: userActivity.loginTime,
        login_location: userActivity.location,
        actions_per_session: userActivity.actions,
        session_duration: userActivity.duration
      }
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true|false, anomaly_score: 0.0-1.0 }
  return result;
};
```

### Burnout Detection

```javascript
// POST /detect-burnout - Analyze user burnout risk
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      workload_metrics: {
        tasks_assigned: 15,
        tasks_completed: 12,
        avg_working_hours: 9.5,
        overtime_hours: 10,
        missed_deadlines: 2
      }
    })
  });
  const result = await response.json();
  // Returns: { burnout_risk: 'low'|'medium'|'high', recommendations: [...] }
  return result;
};
```

### Predictive Insights

```javascript
// POST /predict-project-delay - Predict project completion
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict-project-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.id,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      days_remaining: projectData.daysRemaining,
      team_size: projectData.teamSize,
      avg_task_completion_time: projectData.avgCompletionTime
    })
  });
  const result = await response.json();
  // Returns: { will_delay: true|false, predicted_completion_date: '2026-05-01', confidence: 0.87 }
  return result;
};
```

## Frontend Components

### React Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      // Verify token and fetch user data
      fetchUserData(token);
    }
  }, [token]);

  const login = async (email, password) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    setToken(data.token);
    setUser(data.user);
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const KanbanBoard = () => {
  const { token, user } = useAuth();
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(`http://localhost:5000/api/tasks/user/${user.id}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const categorized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(categorized);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.toUpperCase()}</h3>
          {tasks[column].map(task => (
            <div key={task.id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status}
                onChange={(e) => updateTaskStatus(task.id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in-progress">In Progress</option>
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

### AI Dashboard Component

```javascript
// components/AIDashboard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const AIDashboard = () => {
  const { user, token } = useAuth();
  const [insights, setInsights] = useState({
    riskLevel: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAIInsights();
  }, []);

  const fetchAIInsights = async () => {
    // Fetch risk prediction
    const riskResponse = await fetch('http://localhost:8000/predict-risk', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: user.id })
    });
    const riskData = await riskResponse.json();

    // Fetch burnout detection
    const burnoutResponse = await fetch('http://localhost:8000/detect-burnout', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: user.id })
    });
    const burnoutData = await burnoutResponse.json();

    setInsights({
      riskLevel: riskData.risk_level,
      burnoutRisk: burnoutData.burnout_risk,
      recommendations: burnoutData.recommendations
    });
  };

  return (
    <div className="ai-dashboard">
      <h2>AI Analytics</h2>
      <div className="insight-card">
        <h3>Risk Level</h3>
        <span className={`badge ${insights.riskLevel}`}>
          {insights.riskLevel}
        </span>
      </div>
      <div className="insight-card">
        <h3>Burnout Risk</h3>
        <span className={`badge ${insights.burnoutRisk}`}>
          {insights.burnoutRisk}
        </span>
        {insights.recommendations && (
          <ul>
            {insights.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};

export default AIDashboard;
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const { user, token } = useAuth();

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requireAdmin && user.role !== 'admin') {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracking

```javascript
// components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, onSave }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const remainingSeconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
  };

  const handleSave = () => {
    onSave(taskId, seconds);
    setSeconds(0);
    setIsRunning(false);
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={handleSave}>Save</button>
    </div>
  );
};

export default TimeTracker;
```

## Database Models

### MongoDB User Schema

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['admin', 'user'],
    default: 'user'
  },
  profile: {
    firstName: String,
    lastName: String,
    department: String,
    position: String
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date,
  isActive: {
    type: Boolean,
    default: true
  }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Schema

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: Date,
  timeTracked: {
    type: Number,
    default: 0 // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Schema

```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'feature-request'],
    default: 'general'
  },
  aiClassification: {
    suggestedCategory: String,
    confidence: Number,
    suggestedAssignee: String
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, List, Optional
import joblib
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, tree
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

# Ticket classifier
class TicketData(BaseModel):
    title: str
    description: str
    priority: str

class TicketClassification(BaseModel):
    category: str
    confidence: float
    suggested_assignee: Optional[str]

@app.post("/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketData):
    # Simple rule-based classification (replace with trained model)
    text = f"{ticket.title} {ticket.description}".lower()
    
    if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
        category = 'technical'
        confidence = 0.85
    elif any(word in text for word in ['payment', 'invoice', 'billing']):
        category = 'billing'
        confidence = 0.90
    elif any(word in text for word in ['feature', 'enhancement', 'add']):
        category = 'feature-request'
        confidence = 0.75
    else:
        category = 'general'
        confidence = 0.60
    
    return TicketClassification(
        category=category,
        confidence=confidence,
        suggested_assignee=None
    )

# Risk prediction
class RiskFeatures(BaseModel):
    login_frequency: int
    failed_login_attempts: int
    task_completion_rate: float
    avg_response_time: float
    tickets_raised: int

class RiskPrediction(BaseModel):
    risk_level: str
    risk_score: float

@app.post("/predict-risk", response_model=RiskPrediction)
async def predict_risk(user_id: str, features: RiskFeatures):
    # Calculate risk score based on features
    risk_score = 0.0
    
    # Failed logins contribute to risk
    risk_score += min(features.failed_login_attempts * 0.15, 0.3)
    
    # Low task completion rate increases risk
    if features.task_completion_rate < 0.7:
        risk_score += (0.7 - features.task_completion_rate) * 0.5
    
    # High response time increases risk
    if features.avg_response_time > 240:  # 4 hours
        risk_score += 0.2
    
    # Determine risk level
    if risk_score < 0.3:
        risk_level = 'low'
    elif risk_score < 0.6:
        risk_level = 'medium'
    else:
        risk_level = 'high'
    
    return RiskPrediction(risk_level=risk_level, risk_score=min(risk_score, 1.0))

# Anomaly detection
class ActivityData(BaseModel):
    login_time: str
    login_location: str
    actions_per_session: int
    session_duration: int

class AnomalyDetection(BaseModel):
    is_anomaly: bool
    anomaly_score: float

@app.post("/detect-anomaly", response_model=AnomalyDetection)
async def detect_anomaly(user_id: str, activity_data: ActivityData):
    # Simple threshold-based anomaly detection
    anomaly_score = 0.0
    
    # Unusual session duration
    if activity_data.session_duration > 480:  # 8 hours
        anomaly_score += 0.4
    
    # Unusual number of actions
    if activity_data.actions_per_session > 100:
        anomaly_score += 0.3
    
    is_anomaly = anomaly_score > 0.5
    
    return AnomalyDetection(
        is_anomaly=is_anomaly,
        anomaly_score=min(anomaly_score, 1.0)
    )

# Burnout detection
class WorkloadMetrics(BaseModel):
    tasks_assigned: int
    tasks_completed: int
    avg_working_hours: float
    overtime_hours: float
    missed_deadlines: int

class BurnoutDetection(BaseModel):
    burnout_risk: str
    recommendations: List[str]

@app.post("/detect-burnout", response_model=BurnoutDetection)
async def detect_burnout(user_id: str, workload_metrics: WorkloadMetrics):
    burnout_score = 0.0
    recommendations = []
    
    # High workload
    if workload_metrics.tasks_assigned > 15:
        burnout_score += 0.2
        recommendations.append("Consider redistributing tasks")
    
    # Long working hours
    if workload_metrics.avg_working_hours > 9:
        burnout_score += 0.3
        recommendations.append("Reduce working hours to standard 8 hours")
    
    # Overtime
    if workload_metrics.overtime_hours > 10:
        burnout_score += 0.25
        recommendations.append("Minimize overtime work")
    
    # Missed deadlines
    if workload_metrics.missed_deadlines > 2:
        burnout_score += 0.25
        recommendations.append("Extend deadlines or reduce task load")
    
    if burnout_score < 0.3:
        risk_level = 'low'
    elif burnout_score < 0.6:
        risk_level = 'medium'
    else:
        risk_level = 'high'
    
    if not recommendations:
        recommendations.append("Workload is manageable")
    
    return BurnoutDetection(
        burnout_risk=risk_level,
        recommendations=recommendations
    )

# Project delay prediction
class ProjectMetrics(BaseModel):
    project_id: str
    total_tasks: int
    completed_tasks: int
    days_remaining: int
    team_size: int
    avg_task_completion_time: float

class DelayPrediction(BaseModel):
    will_delay: bool
    predicted_completion_date: str
    confidence: float

@app.post("/predict-project-delay", response_model=DelayPrediction)
async def predict_project_delay(metrics: ProjectMetrics):
    # Calculate completion rate
    completion_rate = metrics.completed_tasks / metrics.total_tasks if metrics.total_tasks > 0 else 0
    remaining_tasks = metrics.total_tasks - metrics.completed_tasks
    
    # Estimate days needed
    days_needed = (remaining_tasks * metrics.avg_task_completion_time) / metrics.team_size
    
    will_delay = days_needed > metrics.days_remaining
    confidence = 0.75 + (0.2 * completion_rate)
    
    # Simple date calculation (use datetime for production
