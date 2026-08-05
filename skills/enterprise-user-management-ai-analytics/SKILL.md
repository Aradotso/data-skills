---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing using React, Node.js, and FastAPI ML service
triggers:
  - how do I set up the enterprise user management system with AI analytics
  - integrate AI-based ticket classification and routing
  - implement user task management with burnout detection
  - create admin dashboard with user analytics
  - set up JWT authentication for enterprise app
  - build kanban board with time tracking
  - configure AI risk prediction and anomaly detection
  - deploy enterprise user management system
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System with AI Analytics, a comprehensive full-stack application that manages users, tasks, and support tickets with intelligent AI-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Secure authentication, role-based access control, and user lifecycle management
- **Task Management**: Kanban boards, time tracking, and progress monitoring
- **Support Ticketing**: Intelligent ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Comprehensive analytics, audit logs, and alert management

The system consists of three main components:
1. **Frontend** (React.js) - User and admin interfaces
2. **Backend** (Node.js) - REST APIs and business logic
3. **ML Service** (FastAPI) - AI/ML models for analytics

## Installation

### Prerequisites

```bash
# Required software
node --version  # v14+ required
python --version  # 3.8+ required
```

### Clone and Setup

```bash
# Clone the repository
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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
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
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
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
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
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
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
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

// GET /api/tasks - Get all tasks
const getTasks = async (token, filters = {}) => {
  const queryParams = new URLSearchParams(filters).toString();
  const response = await fetch(`http://localhost:5000/api/tasks?${queryParams}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status }) // 'todo', 'inProgress', 'done'
  });
  return response.json();
};

// POST /api/tasks/:id/time - Track time
const logTime = async (taskId, timeData, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      duration: timeData.duration, // in seconds
      date: timeData.date
    })
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
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category,
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get tickets
const getTickets = async (token, status = null) => {
  const url = status 
    ? `http://localhost:5000/api/tickets?status=${status}`
    : 'http://localhost:5000/api/tickets';
  
  const response = await fetch(url, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PUT /api/tickets/:id/assign - Assign ticket
const assignTicket = async (ticketId, assigneeId, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}/assign`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ assigneeId })
  });
  return response.json();
};
```

## AI/ML Service API

### Risk Prediction

```javascript
// POST /predict/risk - Predict user risk
const predictUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/predict/risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userData.userId,
      login_frequency: userData.loginFrequency,
      failed_logins: userData.failedLogins,
      tasks_completed: userData.tasksCompleted,
      avg_task_duration: userData.avgTaskDuration,
      active_tickets: userData.activeTickets
    })
  });
  return response.json();
  // Returns: { risk_score: 0.75, risk_level: 'high', factors: [...] }
};
```

### Anomaly Detection

```javascript
// POST /detect/anomaly - Detect anomalous behavior
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/detect/anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: activityData.userId,
      timestamp: activityData.timestamp,
      action: activityData.action,
      ip_address: activityData.ipAddress,
      location: activityData.location,
      device_info: activityData.deviceInfo
    })
  });
  return response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.89, reason: '...' }
};
```

### Burnout Detection

```javascript
// POST /predict/burnout - Analyze burnout risk
const analyzeBurnout = async (workloadData) => {
  const response = await fetch('http://localhost:8000/predict/burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: workloadData.userId,
      hours_worked: workloadData.hoursWorked,
      tasks_count: workloadData.tasksCount,
      overdue_tasks: workloadData.overdueTasks,
      stress_indicators: workloadData.stressIndicators,
      weekend_work: workloadData.weekendWork
    })
  });
  return response.json();
  // Returns: { burnout_risk: 'moderate', score: 0.65, recommendations: [...] }
};
```

### Ticket Classification

```javascript
// POST /classify/ticket - Auto-classify support ticket
const classifyTicket = async (ticketContent) => {
  const response = await fetch('http://localhost:8000/classify/ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketContent.title,
      description: ticketContent.description
    })
  });
  return response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_assignee: 'user_id' }
};
```

### Project Delay Prediction

```javascript
// POST /predict/delay - Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict/delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      avg_completion_rate: projectData.avgCompletionRate,
      team_size: projectData.teamSize,
      deadline: projectData.deadline
    })
  });
  return response.json();
  // Returns: { delay_probability: 0.42, estimated_completion: '2026-05-15', risk_factors: [...] }
};
```

## React Frontend Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      // Verify token and fetch user data
      fetchUserProfile();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUserProfile = async () => {
    try {
      const response = await fetch('http://localhost:5000/api/auth/profile', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = (newToken, userData) => {
    localStorage.setItem('token', newToken);
    setToken(newToken);
    setUser(userData);
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const KanbanBoard = () => {
  const { token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await fetch('http://localhost:5000/api/tasks', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      
      // Group tasks by status
      const grouped = {
        todo: data.filter(t => t.status === 'todo'),
        inProgress: data.filter(t => t.status === 'inProgress'),
        done: data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({ status: newStatus })
      });
      fetchTasks(); // Refresh board
    } catch (error) {
      console.error('Failed to move task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <div className="task-actions">
                {status !== 'todo' && (
                  <button onClick={() => moveTask(task._id, 'todo')}>← Todo</button>
                )}
                {status !== 'inProgress' && (
                  <button onClick={() => moveTask(task._id, 'inProgress')}>
                    {status === 'todo' ? '→ In Progress' : '← In Progress'}
                  </button>
                )}
                {status !== 'done' && (
                  <button onClick={() => moveTask(task._id, 'done')}>→ Done</button>
                )}
              </div>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracking Component

```javascript
// src/components/TimeTracker.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const TimeTracker = ({ taskId }) => {
  const { token } = useContext(AuthContext);
  const [isRunning, setIsRunning] = useState(false);
  const [elapsed, setElapsed] = useState(0);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setElapsed(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const handleStart = () => setIsRunning(true);
  const handlePause = () => setIsRunning(false);
  
  const handleStop = async () => {
    setIsRunning(false);
    if (elapsed > 0) {
      try {
        await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({
            duration: elapsed,
            date: new Date().toISOString()
          })
        });
        setElapsed(0);
      } catch (error) {
        console.error('Failed to log time:', error);
      }
    }
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(elapsed)}</div>
      <div className="time-controls">
        {!isRunning ? (
          <button onClick={handleStart}>Start</button>
        ) : (
          <button onClick={handlePause}>Pause</button>
        )}
        <button onClick={handleStop} disabled={elapsed === 0}>Stop & Save</button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalyticsDashboard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const AIAnalyticsDashboard = () => {
  const { token } = useContext(AuthContext);
  const [analytics, setAnalytics] = useState({
    risks: [],
    anomalies: [],
    burnoutAlerts: [],
    projectDelays: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk predictions
      const riskResponse = await fetch('http://localhost:5000/api/analytics/risks', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const risks = await riskResponse.json();

      // Fetch anomalies
      const anomalyResponse = await fetch('http://localhost:5000/api/analytics/anomalies', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const anomalies = await anomalyResponse.json();

      // Fetch burnout alerts
      const burnoutResponse = await fetch('http://localhost:5000/api/analytics/burnout', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const burnoutAlerts = await burnoutResponse.json();

      setAnalytics({ risks, anomalies, burnoutAlerts });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <section className="analytics-section">
        <h2>Risk Predictions</h2>
        {analytics.risks.map(risk => (
          <div key={risk.user_id} className={`alert alert-${risk.risk_level}`}>
            <strong>{risk.user_name}</strong> - Risk Level: {risk.risk_level}
            <p>Score: {(risk.risk_score * 100).toFixed(1)}%</p>
          </div>
        ))}
      </section>

      <section className="analytics-section">
        <h2>Anomaly Detections</h2>
        {analytics.anomalies.map((anomaly, idx) => (
          <div key={idx} className="alert alert-warning">
            <strong>{anomaly.user_name}</strong> - {anomaly.action}
            <p>{anomaly.reason}</p>
            <small>{new Date(anomaly.timestamp).toLocaleString()}</small>
          </div>
        ))}
      </section>

      <section className="analytics-section">
        <h2>Burnout Alerts</h2>
        {analytics.burnoutAlerts.map(alert => (
          <div key={alert.user_id} className={`alert alert-${alert.burnout_risk}`}>
            <strong>{alert.user_name}</strong> - {alert.burnout_risk} risk
            <ul>
              {alert.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        ))}
      </section>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Backend Model Examples

### User Model (MongoDB)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please provide a name'],
    trim: true
  },
  email: {
    type: String,
    required: [true, 'Please provide an email'],
    unique: true,
    lowercase: true,
    match: [/^\S+@\S+\.\S+$/, 'Please provide a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please provide a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: {
    type: Date
  },
  failedLoginAttempts: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

// Generate JWT token
userSchema.methods.generateAuthToken = function() {
  return jwt.sign(
    { id: this._id, role: this.role },
    process.env.JWT_SECRET,
    { expiresIn: process.env.JWT_EXPIRE || '7d' }
  );
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
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'inProgress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
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
  dueDate: {
    type: Date
  },
  timeTracking: [{
    duration: Number, // in seconds
    date: Date,
    loggedAt: {
      type: Date,
      default: Date.now
    }
  }],
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

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'account', 'general', 'urgent'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'assigned', 'in-progress', 'resolved', 'closed'],
    default: 'open'
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
  aiClassification: {
    suggestedCategory: String,
    suggestedPriority: String,
    confidence: Number
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: {
    type: Date
  }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## ML Service Implementation

### FastAPI Main Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="Enterprise AI Analytics Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load pre-trained models (ensure these exist)
try:
    risk_model = joblib.load('./models/risk_model.pkl')
    anomaly_model = joblib.load('./models/anomaly_model.pkl')
    burnout_model = joblib.load('./models/burnout_model.pkl')
except Exception as e:
    print(f"Warning: Could not load models: {e}")
    risk_model = anomaly_model = burnout_model = None

# Pydantic models
class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: float
    failed_logins: int
    tasks_completed: int
    avg_task_duration: float
    active_tickets: int

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    timestamp: str
    action: str
    ip_address: str
    location: Optional[str] = None
    device_info: Optional[str] = None

class BurnoutPredictionRequest(BaseModel):
    user_id: str
    hours_worked: float
    tasks_count: int
    overdue_tasks: int
    stress_indicators: int
    weekend_work: bool

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

@app.get("/")
def read_root():
    return {"service": "Enterprise AI Analytics", "status": "running"}

@app.post("/predict/risk")
def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior patterns"""
    try:
        # Feature engineering
        features = np.array([[
            request.login_frequency,
            request.failed_logins,
            request.tasks_completed,
            request.avg_task_duration,
            request.active_tickets
        ]])
        
        if risk_model:
            risk_score = risk_model.predict_proba(features)[0][1]
        else:
            # Fallback heuristic
            risk_score = (
                (request.failed_logins * 0.3) +
                (1 / (request.login_frequency + 1) * 0.2) +
                (request.active_tickets * 0.1) +
                (1 / (request.tasks_completed + 1) * 0.4)
            )
        
        risk_level = 'high' if risk_score > 0.7 else 'medium' if risk_score > 0.4 else 'low'
        
        factors = []
        if request.failed_logins > 3:
            factors.append(f"High failed login attempts: {request.failed_logins}")
        if request.login_frequency < 0.3:
            factors.append("Low login frequency")
        if request.tasks_completed < 5:
            factors.append("Low task completion rate")
        
        return {
            "user_id": request.user_id,
            "risk_score": float(risk_score),
            "risk_level": risk_level,
            "
