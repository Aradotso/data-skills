---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket classification, and predictive insights
triggers:
  - "how do I set up the enterprise user management system"
  - "integrate AI analytics into user management"
  - "create a task management dashboard with AI"
  - "implement JWT authentication for user management"
  - "build a ticket classification system with machine learning"
  - "set up Kanban board with time tracking"
  - "configure AI-powered risk detection for users"
  - "deploy user management system with FastAPI ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics. It provides user authentication, task management with Kanban boards, support ticket systems, and ML-based insights including risk prediction, anomaly detection, burnout analysis, and predictive project analytics.

## Project Architecture

The system consists of three main components:

1. **Frontend**: React.js application for user interface
2. **Backend**: Node.js REST API with MongoDB
3. **ML Service**: FastAPI-based machine learning microservice

## Installation & Setup

### Prerequisites

```bash
# Required software
node >= 14.x
python >= 3.8
mongodb >= 4.x
```

### Clone and Install

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env)**:
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**:
```bash
# ml-service/.env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

**Frontend (.env)**:
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

### Running the System

```bash
# Terminal 1: Start MongoDB
mongod --dbpath /path/to/data

# Terminal 2: Start backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 3: Start ML service
cd ml-service
uvicorn main:app --reload
# Runs at http://localhost:8000

# Terminal 4: Start frontend
cd frontend
npm start
# Runs at http://localhost:3000
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
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return await response.json();
};

// Login
const login = async (credentials) => {
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

// Authenticated request helper
const fetchWithAuth = async (url, options = {}) => {
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
// Get all users (admin only)
const getAllUsers = async () => {
  const response = await fetchWithAuth('http://localhost:5000/api/users');
  return await response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const response = await fetchWithAuth(
    `http://localhost:5000/api/users/${userId}`,
    {
      method: 'PUT',
      body: JSON.stringify(updates)
    }
  );
  return await response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const response = await fetchWithAuth(
    `http://localhost:5000/api/users/${userId}`,
    { method: 'DELETE' }
  );
  return await response.json();
};

// Assign role
const assignRole = async (userId, role) => {
  const response = await fetchWithAuth(
    `http://localhost:5000/api/users/${userId}/role`,
    {
      method: 'PATCH',
      body: JSON.stringify({ role }) // 'admin', 'user', 'manager'
    }
  );
  return await response.json();
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData) => {
  const response = await fetchWithAuth('http://localhost:5000/api/tasks', {
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
  return await response.json();
};

// Get user tasks
const getUserTasks = async (userId) => {
  const response = await fetchWithAuth(
    `http://localhost:5000/api/tasks/user/${userId}`
  );
  return await response.json();
};

// Update task status (Kanban)
const updateTaskStatus = async (taskId, status) => {
  const response = await fetchWithAuth(
    `http://localhost:5000/api/tasks/${taskId}/status`,
    {
      method: 'PATCH',
      body: JSON.stringify({ status })
    }
  );
  return await response.json();
};

// Track time on task
const trackTime = async (taskId, timeData) => {
  const response = await fetchWithAuth(
    `http://localhost:5000/api/tasks/${taskId}/time`,
    {
      method: 'POST',
      body: JSON.stringify({
        startTime: timeData.startTime,
        endTime: timeData.endTime,
        duration: timeData.duration // in seconds
      })
    }
  );
  return await response.json();
};
```

### Support Tickets

```javascript
// Create ticket
const createTicket = async (ticketData) => {
  const response = await fetchWithAuth('http://localhost:5000/api/tickets', {
    method: 'POST',
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category // 'technical', 'billing', 'general'
    })
  });
  return await response.json();
};

// Get user tickets
const getUserTickets = async (userId) => {
  const response = await fetchWithAuth(
    `http://localhost:5000/api/tickets/user/${userId}`
  );
  return await response.json();
};

// Update ticket status
const updateTicketStatus = async (ticketId, status) => {
  const response = await fetchWithAuth(
    `http://localhost:5000/api/tickets/${ticketId}/status`,
    {
      method: 'PATCH',
      body: JSON.stringify({ status }) // 'open', 'in-progress', 'resolved', 'closed'
    }
  );
  return await response.json();
};
```

## ML Service API Usage

### AI-Powered Ticket Classification

```javascript
// Classify ticket automatically
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketText.title,
      description: ticketText.description
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.87 }
  return result;
};

// Use in ticket creation workflow
const createAndClassifyTicket = async (ticketData) => {
  // Get AI classification
  const classification = await classifyTicket(ticketData);
  
  // Create ticket with AI suggestions
  const ticket = await createTicket({
    ...ticketData,
    category: classification.category,
    priority: classification.priority,
    aiConfidence: classification.confidence
  });
  
  return ticket;
};
```

### Risk Prediction

```javascript
// Predict user risk score
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ userId })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.23, riskLevel: 'low', factors: [...] }
  return result;
};

// Monitor high-risk users
const monitorHighRiskUsers = async () => {
  const users = await getAllUsers();
  const riskAnalysis = await Promise.all(
    users.map(async (user) => ({
      ...user,
      risk: await predictUserRisk(user.id)
    }))
  );
  
  return riskAnalysis.filter(u => u.risk.riskLevel === 'high');
};
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userActivity.userId,
      loginTime: userActivity.loginTime,
      loginLocation: userActivity.loginLocation,
      activityPattern: userActivity.pattern
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.92, type: 'unusual_location' }
  return result;
};

// Real-time anomaly monitoring
const monitorUserActivity = async (userId, activity) => {
  const anomaly = await detectAnomaly({
    userId,
    loginTime: new Date().toISOString(),
    loginLocation: activity.location,
    pattern: activity.actions
  });
  
  if (anomaly.isAnomaly && anomaly.anomalyScore > 0.8) {
    // Alert admin
    await createAlert({
      type: 'security',
      severity: 'high',
      message: `Anomalous activity detected for user ${userId}`,
      details: anomaly
    });
  }
  
  return anomaly;
};
```

### Burnout Detection

```javascript
// Analyze user burnout risk
const analyzeBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/analyze-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ userId })
  });
  const result = await response.json();
  // Returns: { burnoutScore: 0.68, level: 'moderate', recommendations: [...] }
  return result;
};

// Proactive workload management
const manageWorkload = async (userId) => {
  const burnout = await analyzeBurnout(userId);
  
  if (burnout.burnoutScore > 0.7) {
    // Redistribute tasks
    const userTasks = await getUserTasks(userId);
    const openTasks = userTasks.filter(t => t.status !== 'done');
    
    // Suggest task redistribution
    return {
      warning: true,
      currentLoad: openTasks.length,
      recommendation: `Reduce workload by ${Math.ceil(openTasks.length * 0.3)} tasks`,
      burnoutData: burnout
    };
  }
  
  return { warning: false, burnoutData: burnout };
};
```

### Predictive Project Insights

```javascript
// Predict project delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.id,
      tasks: projectData.tasks,
      teamSize: projectData.teamSize,
      deadline: projectData.deadline
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.34, estimatedDelay: 5, unit: 'days' }
  return result;
};

// Project health monitoring
const monitorProjectHealth = async (projectId) => {
  const project = await getProject(projectId);
  const tasks = await getProjectTasks(projectId);
  
  const prediction = await predictProjectDelay({
    id: projectId,
    tasks: tasks,
    teamSize: project.teamMembers.length,
    deadline: project.deadline
  });
  
  if (prediction.delayProbability > 0.5) {
    // Alert project manager
    await createNotification({
      type: 'project_risk',
      recipientRole: 'manager',
      message: `Project ${projectId} has ${Math.round(prediction.delayProbability * 100)}% risk of delay`,
      prediction
    });
  }
  
  return prediction;
};
```

## Frontend Component Patterns

### Protected Routes with Authentication

```javascript
// ProtectedRoute.js
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }
  
  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import ProtectedRoute from './components/ProtectedRoute';
import AdminDashboard from './pages/AdminDashboard';
import UserDashboard from './pages/UserDashboard';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route 
          path="/admin" 
          element={
            <ProtectedRoute requiredRole="admin">
              <AdminDashboard />
            </ProtectedRoute>
          } 
        />
        <Route 
          path="/dashboard" 
          element={
            <ProtectedRoute>
              <UserDashboard />
            </ProtectedRoute>
          } 
        />
      </Routes>
    </BrowserRouter>
  );
}
```

### Kanban Board Component

```javascript
// KanbanBoard.js
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  
  useEffect(() => {
    loadTasks();
  }, [userId]);
  
  const loadTasks = async () => {
    const userTasks = await getUserTasks(userId);
    const grouped = {
      todo: userTasks.filter(t => t.status === 'todo'),
      inProgress: userTasks.filter(t => t.status === 'in-progress'),
      done: userTasks.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };
  
  const moveTask = async (taskId, newStatus) => {
    await updateTaskStatus(taskId, newStatus);
    await loadTasks();
  };
  
  const TaskCard = ({ task }) => (
    <div className="task-card" draggable onDragEnd={() => {}}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => moveTask(task.id, 
            task.status === 'todo' ? 'in-progress' : 'done'
          )}>
            Move →
          </button>
        )}
      </div>
    </div>
  );
  
  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => <TaskCard key={task.id} task={task} />)}
      </div>
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => <TaskCard key={task.id} task={task} />)}
      </div>
      <div className="column">
        <h3>Done</h3>
        {tasks.done.map(task => <TaskCard key={task.id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracking Component

```javascript
// TimeTracker.js
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const [startTime, setStartTime] = useState(null);
  
  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setElapsedTime(Date.now() - startTime);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning, startTime]);
  
  const startTimer = () => {
    setStartTime(Date.now());
    setIsRunning(true);
  };
  
  const stopTimer = async () => {
    setIsRunning(false);
    const duration = Math.floor(elapsedTime / 1000);
    
    await trackTime(taskId, {
      startTime: new Date(startTime).toISOString(),
      endTime: new Date().toISOString(),
      duration
    });
    
    setElapsedTime(0);
  };
  
  const formatTime = (ms) => {
    const seconds = Math.floor((ms / 1000) % 60);
    const minutes = Math.floor((ms / (1000 * 60)) % 60);
    const hours = Math.floor(ms / (1000 * 60 * 60));
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
  };
  
  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={startTimer}>Start</button>
        ) : (
          <button onClick={stopTimer}>Stop</button>
        )}
      </div>
    </div>
  );
};

export default TimeTracker;
```

### AI Analytics Dashboard

```javascript
// AIAnalyticsDashboard.js
import React, { useState, useEffect } from 'react';

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    risk: null,
    burnout: null,
    recentAnomalies: []
  });
  
  useEffect(() => {
    loadAnalytics();
  }, [userId]);
  
  const loadAnalytics = async () => {
    const [risk, burnout] = await Promise.all([
      predictUserRisk(userId),
      analyzeBurnout(userId)
    ]);
    
    setAnalytics({ risk, burnout, recentAnomalies: [] });
  };
  
  const getRiskColor = (level) => {
    const colors = { low: 'green', medium: 'orange', high: 'red' };
    return colors[level] || 'gray';
  };
  
  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics</h2>
      
      <div className="analytics-card">
        <h3>Risk Assessment</h3>
        <div className="risk-meter">
          <div 
            className="risk-bar" 
            style={{ 
              width: `${analytics.risk?.riskScore * 100}%`,
              backgroundColor: getRiskColor(analytics.risk?.riskLevel)
            }}
          />
        </div>
        <p>Risk Level: <strong>{analytics.risk?.riskLevel}</strong></p>
        {analytics.risk?.factors && (
          <ul>
            {analytics.risk.factors.map((factor, i) => (
              <li key={i}>{factor}</li>
            ))}
          </ul>
        )}
      </div>
      
      <div className="analytics-card">
        <h3>Burnout Analysis</h3>
        <div className="burnout-score">
          <span className="score">{Math.round(analytics.burnout?.burnoutScore * 100)}%</span>
          <span className="level">{analytics.burnout?.level}</span>
        </div>
        {analytics.burnout?.recommendations && (
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {analytics.burnout.recommendations.map((rec, i) => (
                <li key={i}>{rec}</li>
              ))}
            </ul>
          </div>
        )}
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Configuration Best Practices

### Backend MongoDB Schema Examples

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { 
    type: String, 
    enum: ['user', 'admin', 'manager'], 
    default: 'user' 
  },
  profile: {
    firstName: String,
    lastName: String,
    department: String,
    phoneNumber: String
  },
  isActive: { type: Boolean, default: true },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
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
  assignedTo: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User', 
    required: true 
  },
  createdBy: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User' 
  },
  dueDate: Date,
  completedAt: Date,
  timeTracking: [{
    startTime: Date,
    endTime: Date,
    duration: Number // in seconds
  }],
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### JWT Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authenticate = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);
    
    if (!user || !user.isActive) {
      return res.status(401).json({ error: 'Invalid authentication' });
    }
    
    req.user = user;
    req.userId = user._id;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const requireRole = (roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
};

module.exports = { authenticate, requireRole };
```

## ML Service Python Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    userId: str

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    loginLocation: str
    activityPattern: List[str]

class BurnoutAnalysisRequest(BaseModel):
    userId: str

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """AI-powered ticket classification"""
    try:
        # Combine text features
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification (replace with actual ML model)
        categories = {
            'technical': ['bug', 'error', 'crash', 'not working', 'broken'],
            'billing': ['invoice', 'payment', 'charge', 'refund', 'subscription'],
            'general': ['question', 'how to', 'help', 'support']
        }
        
        category = 'general'
        max_matches = 0
        
        for cat, keywords in categories.items():
            matches = sum(1 for kw in keywords if kw in text)
            if matches > max_matches:
                max_matches = matches
                category = cat
        
        # Determine priority based on urgency keywords
        priority = 'medium'
        if any(word in text for word in ['urgent', 'critical', 'emergency', 'asap']):
            priority = 'high'
        elif any(word in text for word in ['minor', 'low priority', 'when possible']):
            priority = 'low'
        
        return {
            'category': category,
            'priority': priority,
            'confidence': 0.75 + (max_matches * 0.05)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior patterns"""
    try:
        # In production, fetch user data from database
        # Here we simulate risk calculation
        
        # Factors: login frequency, task completion rate, ticket history
        risk_score = np.random.beta(2, 5)  # Replace with actual model
        
        risk_level = 'low'
        if risk_score > 0.7:
            risk_level = 'high'
        elif risk_score > 0.4:
            risk_level = 'medium'
        
        factors = []
        if risk_score > 0.5:
            factors.append('Irregular login patterns')
        if risk_score > 0.6:
            factors.append('Low task completion rate')
        
        return {
            'riskScore': round(risk_score, 2),
            'riskLevel': risk_level,
            'factors': factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    try:
        # Parse login time
        login_dt = datetime.fromisoformat(request.loginTime.replace('Z', '+00:00'))
        
        # Check for unusual login times (outside 6 AM - 10 PM)
        hour = login_dt.hour
        unusual_time = hour < 6 or hour > 22
        
        # Check for unusual location (simplified)
        unusual_location = 'unknown' in request.loginLocation.lower()
        
        # Check activity pattern anomalies
        unusual_activity = len(request.activityPattern) > 50  # Too many actions
        
        is_anomaly = unusual_time or unusual_location or unusual_activity
        anomaly_score = (
            (0.4 if unusual_time else 0) +
            (0.4 if unusual_location else 0) +
            (0.3 if unusual_activity else 0)
        )
        
        anomaly_type = []
        
