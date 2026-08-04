---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user management dashboard with machine learning"
  - "add risk detection to user system"
  - "build admin panel with AI insights"
  - "integrate ticket classification ML service"
  - "deploy user management with FastAPI analytics"
  - "configure JWT authentication for user system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack JavaScript application that combines traditional user/task management with AI-powered analytics. The system provides role-based access control, Kanban task boards, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and ticket classification.

**Architecture:**
- **Frontend**: React.js dashboard (port 3000)
- **Backend**: Node.js REST API with MongoDB (port 5000)
- **ML Service**: FastAPI with scikit-learn and River (port 8000)

## Installation

### Prerequisites

```bash
# Required
node --version  # v14+
python --version  # 3.8+
mongod --version  # MongoDB 4.4+
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

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
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
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=info
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
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Key API Endpoints

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
      role: 'user' // or 'admin'
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
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

```javascript
// GET /api/users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
};

// PUT /api/users/:id
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

// DELETE /api/users/:id
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Task Management

```javascript
// POST /api/tasks
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
      status: 'todo', // 'todo', 'in-progress', 'done'
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};

// GET /api/tasks/user/:userId
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// PATCH /api/tasks/:id/status
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
// POST /api/tickets
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return await response.json();
};

// GET /api/tickets
const getTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### AI/ML Analytics Endpoints

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText, token) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      text: ticketText,
      subject: ticketText.substring(0, 50)
    })
  });
  return await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.85 }
};

// POST /api/ml/detect-risk
const detectUserRisk = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-risk', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ userId })
  });
  return await response.json();
  // Returns: { riskScore: 0.72, factors: ['high_failed_logins', 'unusual_activity'] }
};

// POST /api/ml/detect-burnout
const detectBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ userId })
  });
  return await response.json();
  // Returns: { burnoutScore: 0.68, workloadHours: 65, recommendations: [...] }
};

// POST /api/ml/predict-project
const predictProjectDelay = async (projectData, token) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-project', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      projectId: projectData.id,
      completionPercentage: projectData.completion,
      daysRemaining: projectData.daysLeft,
      teamSize: projectData.teamSize
    })
  });
  return await response.json();
  // Returns: { delayProbability: 0.45, estimatedDelay: 5 }
};

// POST /api/ml/detect-anomaly
const detectAnomaly = async (activityData, token) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: activityData.userId,
      loginTime: activityData.timestamp,
      ipAddress: activityData.ip,
      location: activityData.location
    })
  });
  return await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.89, reason: 'unusual_location' }
};
```

## React Component Patterns

### Protected Route with Authentication

```javascript
// components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requireAdmin && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
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
    fetchTasks();
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
      <div key={task._id} className="task-card">
        <h4>{task.title}</h4>
        <p>{task.description}</p>
        <select onChange={(e) => onMove(task._id, e.target.value)} value={targetStatus}>
          <option value="todo">To Do</option>
          <option value="in-progress">In Progress</option>
          <option value="done">Done</option>
        </select>
      </div>
    ))}
  </div>
);

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutScore: null,
    anomalies: []
  });
  const token = localStorage.getItem('token');

  useEffect(() => {
    loadAnalytics();
  }, [userId]);

  const loadAnalytics = async () => {
    try {
      const [risk, burnout] = await Promise.all([
        fetch('http://localhost:8000/api/ml/detect-risk', {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({ userId })
        }).then(r => r.json()),
        
        fetch('http://localhost:8000/api/ml/detect-burnout', {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({ userId })
        }).then(r => r.json())
      ]);

      setAnalytics({
        riskScore: risk.riskScore,
        burnoutScore: burnout.burnoutScore,
        recommendations: burnout.recommendations
      });
    } catch (error) {
      console.error('Analytics loading failed:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div className={`score ${analytics.riskScore > 0.7 ? 'high' : 'low'}`}>
          {(analytics.riskScore * 100).toFixed(0)}%
        </div>
      </div>
      
      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <div className={`score ${analytics.burnoutScore > 0.6 ? 'high' : 'low'}`}>
          {(analytics.burnoutScore * 100).toFixed(0)}%
        </div>
      </div>

      {analytics.recommendations && (
        <div className="recommendations">
          <h4>Recommendations</h4>
          <ul>
            {analytics.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Backend API Implementation

### User Model (MongoDB Schema)

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  failedLoginAttempts: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now }
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

### Task Model

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Authentication Middleware

```javascript
// middleware/auth.js
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
    return res.status(401).json({ message: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ message: 'Admin access required' });
  }
};

module.exports = { protect, adminOnly };
```

### Task Routes

```javascript
// routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect, adminOnly } = require('../middleware/auth');

// Create task (admin only)
router.post('/', protect, adminOnly, async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user._id
    });
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Get user tasks
router.get('/user/:userId', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Track time
router.patch('/:id/time', protect, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    task.timeTracked += req.body.seconds;
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation (FastAPI)

### Ticket Classification

```python
# ml-service/services/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import pickle
import os

class TicketClassifier:
    def __init__(self):
        self.vectorizer = TfidfVectorizer(max_features=1000)
        self.model = MultinomialNB()
        self.categories = ['technical', 'billing', 'general', 'urgent']
        self.load_model()
    
    def load_model(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            with open(f'{model_path}/ticket_classifier.pkl', 'rb') as f:
                self.model = pickle.load(f)
            with open(f'{model_path}/vectorizer.pkl', 'rb') as f:
                self.vectorizer = pickle.load(f)
        except FileNotFoundError:
            print("Model not found, using untrained model")
    
    def classify(self, text):
        features = self.vectorizer.transform([text])
        prediction = self.model.predict(features)[0]
        proba = self.model.predict_proba(features)[0]
        
        return {
            'category': self.categories[prediction],
            'confidence': float(max(proba))
        }
```

### Risk Detection

```python
# ml-service/services/risk_detector.py
from river import anomaly
import numpy as np

class RiskDetector:
    def __init__(self):
        self.model = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)
    
    def calculate_risk(self, user_data):
        features = {
            'failed_logins': user_data.get('failedLoginAttempts', 0),
            'days_since_last_login': user_data.get('daysSinceLastLogin', 0),
            'unusual_hours': user_data.get('unusualHourLogins', 0),
            'location_changes': user_data.get('locationChanges', 0)
        }
        
        score = self.model.score_one(features)
        self.model.learn_one(features)
        
        risk_factors = []
        if features['failed_logins'] > 3:
            risk_factors.append('high_failed_logins')
        if features['unusual_hours'] > 5:
            risk_factors.append('unusual_activity')
        if features['location_changes'] > 3:
            risk_factors.append('multiple_locations')
        
        return {
            'riskScore': min(score / 100, 1.0),
            'factors': risk_factors
        }
```

### Burnout Detection

```python
# ml-service/services/burnout_detector.py
class BurnoutDetector:
    def analyze(self, user_id, task_data):
        total_hours = sum(task.get('timeTracked', 0) for task in task_data) / 3600
        task_count = len(task_data)
        overdue_tasks = len([t for t in task_data if t.get('isOverdue', False)])
        
        # Simple scoring algorithm
        hours_score = min(total_hours / 60, 1.0)  # Normalize to 60 hours/week
        task_score = min(task_count / 30, 1.0)  # Normalize to 30 tasks
        overdue_score = min(overdue_tasks / 10, 1.0)  # Normalize to 10 overdue
        
        burnout_score = (hours_score * 0.5 + task_score * 0.3 + overdue_score * 0.2)
        
        recommendations = []
        if hours_score > 0.7:
            recommendations.append("Reduce working hours")
        if task_score > 0.7:
            recommendations.append("Delegate some tasks")
        if overdue_score > 0.5:
            recommendations.append("Prioritize overdue tasks")
        
        return {
            'burnoutScore': burnout_score,
            'workloadHours': total_hours,
            'taskCount': task_count,
            'recommendations': recommendations
        }
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from services.ticket_classifier import TicketClassifier
from services.risk_detector import RiskDetector
from services.burnout_detector import BurnoutDetector
import os

app = FastAPI(title="ML Analytics Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize services
ticket_classifier = TicketClassifier()
risk_detector = RiskDetector()
burnout_detector = BurnoutDetector()

class TicketRequest(BaseModel):
    text: str
    subject: str

class RiskRequest(BaseModel):
    userId: str
    failedLoginAttempts: int = 0
    daysSinceLastLogin: int = 0
    unusualHourLogins: int = 0
    locationChanges: int = 0

class BurnoutRequest(BaseModel):
    userId: str
    tasks: list

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    result = ticket_classifier.classify(request.text)
    return result

@app.post("/api/ml/detect-risk")
async def detect_risk(request: RiskRequest):
    user_data = request.dict()
    result = risk_detector.calculate_risk(user_data)
    return result

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    result = burnout_detector.analyze(request.userId, request.tasks)
    return result

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Configuration

### Environment Variables

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_min_32_chars
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000

# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000

# ml-service/.env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=info
```

### MongoDB Connection

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

## Common Patterns

### Time Tracking Implementation

```javascript
// frontend/hooks/useTimeTracker.js
import { useState, useEffect, useRef } from 'react';

export const useTimeTracker = (taskId) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      clearInterval(intervalRef.current);
    }
    return () => clearInterval(intervalRef.current);
  }, [isRunning]);

  const start = () => setIsRunning(true);
  const stop = () => setIsRunning(false);
  
  const save = async () => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ seconds })
    });
    setSeconds(0);
  };

  return { seconds, isRunning, start, stop, save };
};
```

### Real-time Notifications

```javascript
// frontend/hooks/useNotifications.js
import { useState, useEffect } from 'react';
import io from 'socket.io-client';

export const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);
  const [socket, setSocket] = useState(null);

  useEffect(() => {
    const newSocket = io(process.env.REACT_APP_API_URL);
    setSocket(newSocket);

    newSocket.emit('join', userId);
    
    newSocket.on('notification', (notification) => {
      setNotifications(prev => [notification, ...prev]);
    });

    return () => newSocket.close();
  }, [userId]);

  const markAsRead = (notificationId) => {
    setNotifications(prev => 
      prev.map(n => n.id === notificationId ? { ...n, read: true } : n)
    );
  };

  return { notifications, markAsRead };
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection string
echo $MONGODB_URI
```

### JWT Token Expiration

```javascript
// frontend/utils/api.js
const refreshToken =
