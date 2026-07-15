---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task management, and support ticket automation
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "configure risk detection and anomaly detection"
  - "build admin panel with user management"
  - "integrate AI ticket classification system"
  - "add burnout detection to user system"
  - "set up kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application combining user/task management with machine learning capabilities. It provides:

- **User Management**: Role-based access control, authentication, task assignment
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, ticket classification
- **Task Tracking**: Kanban boards, time tracking, progress monitoring
- **Support System**: Smart ticket routing and management

**Stack**: React.js frontend, Node.js backend, FastAPI ML service, MongoDB database, JWT authentication

## Installation

### Prerequisites

```bash
# Ensure you have installed:
# - Node.js (v14+)
# - Python (v3.8+)
# - MongoDB
```

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

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
NODE_ENV=development
```

Start backend:

```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
API_HOST=0.0.0.0
API_PORT=8000
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Runs at http://localhost:3000
```

## Architecture

```
frontend/          → React.js UI
  ├── src/
  │   ├── components/    → Reusable components
  │   ├── pages/         → Admin/User dashboards
  │   ├── services/      → API calls
  │   └── utils/         → Helpers
backend/           → Node.js REST API
  ├── controllers/       → Request handlers
  ├── models/           → MongoDB schemas
  ├── routes/           → API endpoints
  └── middleware/       → Auth, validation
ml-service/        → FastAPI ML endpoints
  ├── models/           → Trained ML models
  ├── services/         → AI logic
  └── routes/           → ML API routes
```

## Backend API Usage

### Authentication

```javascript
// Register new user
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

// Login
const login = async (email, password) => {
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

// Authenticated request helper
const authenticatedFetch = async (url, options = {}) => {
  const token = localStorage.getItem('token');
  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async () => {
  const response = await authenticatedFetch('http://localhost:5000/api/users');
  return response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const response = await authenticatedFetch(
    `http://localhost:5000/api/users/${userId}`,
    {
      method: 'PUT',
      body: JSON.stringify(updates)
    }
  );
  return response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const response = await authenticatedFetch(
    `http://localhost:5000/api/users/${userId}`,
    { method: 'DELETE' }
  );
  return response.json();
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData) => {
  const response = await authenticatedFetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// Update task status
const updateTaskStatus = async (taskId, status) => {
  const response = await authenticatedFetch(
    `http://localhost:5000/api/tasks/${taskId}/status`,
    {
      method: 'PATCH',
      body: JSON.stringify({ status })
    }
  );
  return response.json();
};

// Get user tasks
const getUserTasks = async (userId) => {
  const response = await authenticatedFetch(
    `http://localhost:5000/api/tasks/user/${userId}`
  );
  return response.json();
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const response = await authenticatedFetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// Get user tickets
const getUserTickets = async () => {
  const response = await authenticatedFetch('http://localhost:5000/api/tickets/user');
  return response.json();
};

// Update ticket (Admin)
const updateTicket = async (ticketId, updates) => {
  const response = await authenticatedFetch(
    `http://localhost:5000/api/tickets/${ticketId}`,
    {
      method: 'PUT',
      body: JSON.stringify({
        status: updates.status,
        assignedTo: updates.assignedTo,
        response: updates.response
      })
    }
  );
  return response.json();
};
```

## ML Service API Usage

### AI Ticket Classification

```javascript
// Classify ticket and route automatically
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: 'Login issues'
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_assignee: 'tech-team' }
  return data;
};
```

### Risk Detection

```javascript
// Analyze user behavior for risk
const analyzeUserRisk = async (userId, behaviorData) => {
  const response = await fetch('http://localhost:8000/ml/risk-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      login_frequency: behaviorData.loginFrequency,
      failed_logins: behaviorData.failedLogins,
      access_patterns: behaviorData.accessPatterns,
      unusual_hours: behaviorData.unusualHours
    })
  });
  const data = await response.json();
  // Returns: { risk_score: 0.75, risk_level: 'high', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: activityData.userId,
      actions: activityData.actions,
      timestamp: activityData.timestamp,
      ip_address: activityData.ipAddress,
      device_info: activityData.deviceInfo
    })
  });
  const data = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.85, reason: 'Unusual access pattern' }
  return data;
};
```

### Burnout Detection

```javascript
// Analyze workload for burnout risk
const detectBurnout = async (userId, workloadData) => {
  const response = await fetch('http://localhost:8000/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      tasks_completed: workloadData.tasksCompleted,
      tasks_pending: workloadData.tasksPending,
      avg_working_hours: workloadData.avgWorkingHours,
      overtime_hours: workloadData.overtimeHours,
      task_completion_rate: workloadData.taskCompletionRate
    })
  });
  const data = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.82, recommendations: [...] }
  return data;
};
```

### Predictive Project Insights

```javascript
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      tasks_total: projectData.tasksTotal,
      tasks_completed: projectData.tasksCompleted,
      days_remaining: projectData.daysRemaining,
      team_size: projectData.teamSize,
      complexity_score: projectData.complexityScore
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.65, predicted_days_delay: 5, factors: [...] }
  return data;
};
```

## React Frontend Patterns

### Protected Route Component

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('userRole');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && userRole !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await authenticatedFetch(
      `http://localhost:5000/api/tasks/user/${userId}`
    );
    const data = await response.json();
    
    const categorized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    
    setTasks(categorized);
  };

  const moveTask = async (taskId, newStatus) => {
    await updateTaskStatus(taskId, newStatus);
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} 
              onMove={(id) => moveTask(id, 'in-progress')} />
      <Column title="In Progress" tasks={tasks.inProgress} 
              onMove={(id) => moveTask(id, 'done')} />
      <Column title="Done" tasks={tasks.done} />
    </div>
  );
};

const Column = ({ title, tasks, onMove }) => (
  <div className="kanban-column">
    <h3>{title}</h3>
    {tasks.map(task => (
      <TaskCard key={task._id} task={task} onMove={onMove} />
    ))}
  </div>
);

export default KanbanBoard;
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isActive, setIsActive] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isActive) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else if (!isActive && seconds !== 0) {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isActive, seconds]);

  const toggle = () => setIsActive(!isActive);

  const reset = async () => {
    // Save time to backend
    await authenticatedFetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'POST',
      body: JSON.stringify({ timeSpent: seconds })
    });
    setSeconds(0);
    setIsActive(false);
  };

  const formatTime = (secs) => {
    const hrs = Math.floor(secs / 3600);
    const mins = Math.floor((secs % 3600) / 60);
    const sec = secs % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${sec.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={toggle}>{isActive ? 'Pause' : 'Start'}</button>
      <button onClick={reset}>Save & Reset</button>
    </div>
  );
};

export default TimeTracker;
```

### AI Insights Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const AIInsightsDashboard = ({ userId }) => {
  const [insights, setInsights] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    // Fetch user activity data
    const activity = await authenticatedFetch(
      `http://localhost:5000/api/users/${userId}/activity`
    ).then(r => r.json());

    // Get risk analysis
    const risk = await fetch('http://localhost:8000/ml/risk-detection', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        user_id: userId,
        login_frequency: activity.loginFrequency,
        failed_logins: activity.failedLogins,
        access_patterns: activity.accessPatterns,
        unusual_hours: activity.unusualHours
      })
    }).then(r => r.json());

    // Get burnout analysis
    const burnout = await fetch('http://localhost:8000/ml/burnout-detection', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        user_id: userId,
        tasks_completed: activity.tasksCompleted,
        tasks_pending: activity.tasksPending,
        avg_working_hours: activity.avgWorkingHours,
        overtime_hours: activity.overtimeHours,
        task_completion_rate: activity.taskCompletionRate
      })
    }).then(r => r.json());

    setInsights({
      riskScore: risk.risk_score,
      burnoutRisk: burnout.burnout_risk,
      anomalies: activity.anomalies || []
    });
  };

  return (
    <div className="ai-insights">
      <div className="insight-card">
        <h3>Security Risk</h3>
        <div className={`risk-meter ${getRiskLevel(insights.riskScore)}`}>
          {insights.riskScore ? `${(insights.riskScore * 100).toFixed(1)}%` : 'N/A'}
        </div>
      </div>
      
      <div className="insight-card">
        <h3>Burnout Risk</h3>
        <div className={`burnout-indicator ${insights.burnoutRisk}`}>
          {insights.burnoutRisk || 'N/A'}
        </div>
      </div>
      
      {insights.anomalies.length > 0 && (
        <div className="insight-card">
          <h3>Detected Anomalies</h3>
          <ul>
            {insights.anomalies.map((anomaly, idx) => (
              <li key={idx}>{anomaly.description}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

const getRiskLevel = (score) => {
  if (!score) return 'unknown';
  if (score < 0.3) return 'low';
  if (score < 0.7) return 'medium';
  return 'high';
};

export default AIInsightsDashboard;
```

## Configuration

### MongoDB Schema Examples

**User Schema** (backend/models/User.js):
```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: String,
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  isActive: { type: Boolean, default: true }
});

module.exports = mongoose.model('User', userSchema);
```

**Task Schema** (backend/models/Task.js):
```javascript
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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
  timeSpent: { type: Number, default: 0 }, // seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

**Ticket Schema** (backend/models/Ticket.js):
```javascript
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  category: String,
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  responses: [{
    message: String,
    by: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    timestamp: { type: Date, default: Date.now }
  }],
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'No authentication token' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

## Common Patterns

### Service Layer Pattern (Backend)

```javascript
// backend/services/taskService.js
const Task = require('../models/Task');

class TaskService {
  async createTask(taskData, creatorId) {
    const task = new Task({
      ...taskData,
      createdBy: creatorId
    });
    await task.save();
    return task;
  }

  async getUserTasks(userId) {
    return Task.find({ assignedTo: userId })
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
  }

  async updateTaskStatus(taskId, status, userId) {
    const task = await Task.findById(taskId);
    
    if (!task) {
      throw new Error('Task not found');
    }

    if (task.assignedTo.toString() !== userId) {
      throw new Error('Unauthorized');
    }

    task.status = status;
    task.updatedAt = new Date();
    await task.save();
    
    return task;
  }

  async getTaskAnalytics(userId) {
    const tasks = await Task.find({ assignedTo: userId });
    
    return {
      total: tasks.length,
      todo: tasks.filter(t => t.status === 'todo').length,
      inProgress: tasks.filter(t => t.status === 'in-progress').length,
      done: tasks.filter(t => t.status === 'done').length,
      totalTimeSpent: tasks.reduce((sum, t) => sum + t.timeSpent, 0)
    };
  }
}

module.exports = new TaskService();
```

### API Route Pattern

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { authMiddleware } = require('../middleware/auth');
const taskService = require('../services/taskService');

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = await taskService.createTask(req.body, req.user.id);
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get user tasks
router.get('/user/:userId', authMiddleware, async (req, res) => {
  try {
    const tasks = await taskService.getUserTasks(req.params.userId);
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:taskId/status', authMiddleware, async (req, res) => {
  try {
    const task = await taskService.updateTaskStatus(
      req.params.taskId,
      req.body.status,
      req.user.id
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### ML Model Integration Pattern (Python)

```python
# ml-service/services/risk_detection.py
from sklearn.ensemble import IsolationForest
import numpy as np
import joblib
import os

class RiskDetectionService:
    def __init__(self):
        model_path = os.getenv('MODEL_PATH', './models')
        self.model_file = f"{model_path}/risk_model.pkl"
        self.model = self._load_or_create_model()
    
    def _load_or_create_model(self):
        if os.path.exists(self.model_file):
            return joblib.load(self.model_file)
        else:
            # Create new model
            model = IsolationForest(contamination=0.1, random_state=42)
            return model
    
    def analyze_risk(self, user_data):
        features = self._extract_features(user_data)
        
        # Predict anomaly (-1 for anomaly, 1 for normal)
        prediction = self.model.predict([features])[0]
        anomaly_score = self.model.score_samples([features])[0]
        
        # Convert to risk score (0-1)
        risk_score = max(0, min(1, (1 - anomaly_score) / 2))
        
        risk_level = self._get_risk_level(risk_score)
        factors = self._identify_risk_factors(user_data)
        
        return {
            'risk_score': float(risk_score),
            'risk_level': risk_level,
            'is_anomaly': prediction == -1,
            'factors': factors
        }
    
    def _extract_features(self, user_data):
        return np.array([
            user_data.get('login_frequency', 0),
            user_data.get('failed_logins', 0),
            len(user_data.get('access_patterns', [])),
            user_data.get('unusual_hours', 0)
        ])
    
    def _get_risk_level(self, score):
        if score < 0.3:
            return 'low'
        elif score < 0.7:
            return 'medium'
        else:
            return 'high'
    
    def _identify_risk_factors(self, user_data):
        factors = []
        if user_data.get('failed_logins', 0) > 3:
            factors.append('Multiple failed login attempts')
        if user_data.get('unusual_hours', 0) > 5:
            factors.append('Unusual access hours')
        return factors

risk_service = RiskDetectionService()
```

### FastAPI Endpoint Pattern

```python
# ml-service/routes/ml_routes.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from services.risk_detection import risk_service
from services.burnout_detection import burnout_service
from services.ticket_classifier import ticket_classifier

router = APIRouter(prefix="/ml", tags=["ML"])

class RiskAnalysisRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    access_patterns: list
    unusual_hours: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_working_hours: float
    overtime_hours: float
    task_completion_rate: float

class TicketClassificationRequest(BaseModel):
    text: str
    subject: str

@router.post("/risk-detection")
async def analyze_risk(request: RiskAnalysisRequest):
    try:
        result = risk_service.analyze_risk(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/burnout-detection")
async def detect_burnout(request: BurnoutAnalysisRequest):
    try:
        result = burnout_service.analyze(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        result = ticket_classifier.classify(
            request.text, 
            request.subject
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Troubleshooting

### JWT Token Issues

```javascript
// Check token validity
const verifyToken = () => {
  const token = localStorage.getItem('token');
  if (!token) return false;
  
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const isExpired = payload.exp * 1000 < Date.now();
    
    if (isExpired) {
      localStorage.removeItem('token');
      return false;
    }
    return true;
  } catch {
    return
