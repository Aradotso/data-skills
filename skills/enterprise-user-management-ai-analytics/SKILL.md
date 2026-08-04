---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, ticket management, and predictive insights
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - create a task management dashboard with AI insights
  - implement JWT authentication for user management
  - build anomaly detection for user behavior
  - set up kanban board with time tracking
  - configure AI-based ticket classification system
  - deploy user management system with ML service
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript/Node.js application that combines traditional user and task management with AI-powered insights. The system provides role-based access control, Kanban-style task boards, support ticket management, and ML-driven features including risk prediction, anomaly detection, burnout analysis, and ticket classification.

**Key Components:**
- **Frontend**: React.js application with user/admin dashboards
- **Backend**: Node.js/Express REST API with JWT authentication
- **ML Service**: FastAPI-based AI service using scikit-learn and River for online learning
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Ensure you have installed:
node --version  # v14+ required
npm --version
python --version  # 3.8+ required
pip --version
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend server
npm start
# Server runs at http://localhost:5000
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
# Service runs at http://localhost:8000
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
# Application runs at http://localhost:3000
```

## Core API Endpoints

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
  // Store token
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
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
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
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      status: 'todo', // 'todo', 'inprogress', 'done'
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks - Get tasks
const getTasks = async (filters, token) => {
  const params = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tasks?${params}`, {
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
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status })
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
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get tickets
const getTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## AI/ML Service API

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.85 }
  return data;
};
```

### Risk Prediction

```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: userId
    })
  });
  const data = await response.json();
  // Returns: { riskScore: 0.65, riskLevel: 'medium', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      loginTime: userData.loginTime,
      loginLocation: userData.loginLocation,
      activityPattern: userData.activityPattern
    })
  });
  const data = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.92, reason: '...' }
  return data;
};
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: userId,
      timeRange: '30d' // days to analyze
    })
  });
  const data = await response.json();
  // Returns: { burnoutScore: 0.78, indicators: [...], recommendations: [...] }
  return data;
};
```

## React Component Patterns

### Authentication Context

```javascript
// context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      fetchCurrentUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchCurrentUser = async () => {
    try {
      const response = await fetch('http://localhost:5000/api/auth/me', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    
    if (data.token) {
      localStorage.setItem('token', data.token);
      setToken(data.token);
      setUser(data.user);
      return { success: true };
    }
    return { success: false, error: data.message };
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
// components/KanbanBoard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const KanbanBoard = () => {
  const { token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch('http://localhost:5000/api/tasks', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inprogress: data.filter(t => t.status === 'inprogress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
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

  const renderColumn = (status, title) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      {tasks[status].map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <select
            value={task.status}
            onChange={(e) => moveTask(task._id, e.target.value)}
          >
            <option value="todo">To Do</option>
            <option value="inprogress">In Progress</option>
            <option value="done">Done</option>
          </select>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('inprogress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalyticsDashboard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const AIAnalyticsDashboard = () => {
  const { token, user } = useContext(AuthContext);
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutScore: null,
    anomalies: []
  });

  useEffect(() => {
    if (user?.role === 'admin') {
      fetchOrgAnalytics();
    } else {
      fetchUserAnalytics();
    }
  }, [user]);

  const fetchUserAnalytics = async () => {
    // Fetch burnout score
    const burnoutResponse = await fetch('http://localhost:8000/api/ml/detect-burnout', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ userId: user._id, timeRange: '30d' })
    });
    const burnoutData = await burnoutResponse.json();

    setAnalytics(prev => ({ ...prev, burnoutScore: burnoutData.burnoutScore }));
  };

  const fetchOrgAnalytics = async () => {
    // Fetch organization-wide risk metrics
    const riskResponse = await fetch('http://localhost:8000/api/ml/org-risk-analysis', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const riskData = await riskResponse.json();

    // Fetch recent anomalies
    const anomalyResponse = await fetch('http://localhost:8000/api/ml/recent-anomalies', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const anomalyData = await anomalyResponse.json();

    setAnalytics({
      riskScore: riskData.avgRiskScore,
      anomalies: anomalyData.anomalies
    });
  };

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics</h2>
      
      {user?.role === 'admin' ? (
        <>
          <div className="metric-card">
            <h3>Organization Risk Score</h3>
            <p className="score">{analytics.riskScore?.toFixed(2) || 'Loading...'}</p>
          </div>
          
          <div className="anomaly-list">
            <h3>Recent Anomalies</h3>
            {analytics.anomalies.map((anomaly, idx) => (
              <div key={idx} className="anomaly-item">
                <span>{anomaly.userId}</span>
                <span>{anomaly.reason}</span>
                <span>{new Date(anomaly.timestamp).toLocaleString()}</span>
              </div>
            ))}
          </div>
        </>
      ) : (
        <div className="metric-card">
          <h3>Your Burnout Risk</h3>
          <p className="score">{analytics.burnoutScore?.toFixed(2) || 'Loading...'}</p>
          <p className="description">
            {analytics.burnoutScore > 0.7 ? 'High - Consider taking a break' :
             analytics.burnoutScore > 0.4 ? 'Moderate - Monitor your workload' :
             'Low - Keep up the good work'}
          </p>
        </div>
      )}
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Backend Middleware Patterns

### JWT Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized, no token' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized, token failed' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized` 
      });
    }
    next();
  };
};

module.exports = { protect, authorize };
```

### Using Middleware

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const { 
  getUsers, 
  getUser, 
  updateUser, 
  deleteUser 
} = require('../controllers/userController');

router.get('/', protect, authorize('admin'), getUsers);
router.get('/:id', protect, getUser);
router.put('/:id', protect, authorize('admin'), updateUser);
router.delete('/:id', protect, authorize('admin'), deleteUser);

module.exports = router;
```

## ML Service Python Patterns

### Ticket Classification Model

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import pickle
import os

class TicketClassifier:
    def __init__(self, model_path='./models/ticket_classifier.pkl'):
        self.model_path = model_path
        self.model = None
        self.load_or_initialize()
    
    def load_or_initialize(self):
        if os.path.exists(self.model_path):
            with open(self.model_path, 'rb') as f:
                self.model = pickle.load(f)
        else:
            # Initialize new model
            self.model = Pipeline([
                ('tfidf', TfidfVectorizer(max_features=1000)),
                ('clf', MultinomialNB())
            ])
    
    def train(self, texts, labels):
        self.model.fit(texts, labels)
        self.save()
    
    def predict(self, text):
        category = self.model.predict([text])[0]
        proba = self.model.predict_proba([text])[0]
        confidence = max(proba)
        
        return {
            'category': category,
            'confidence': float(confidence)
        }
    
    def save(self):
        os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
        with open(self.model_path, 'wb') as f:
            pickle.dump(self.model, f)
```

### Anomaly Detection with River

```python
# ml-service/models/anomaly_detector.py
from river import anomaly
from river import preprocessing
import json

class AnomalyDetector:
    def __init__(self):
        # Use Half-Space Trees for online anomaly detection
        self.scaler = preprocessing.StandardScaler()
        self.detector = anomaly.HalfSpaceTrees(seed=42)
    
    def extract_features(self, user_activity):
        """Extract numerical features from user activity"""
        return {
            'login_hour': user_activity.get('loginTime', 0),
            'tasks_completed': user_activity.get('tasksCompleted', 0),
            'avg_task_time': user_activity.get('avgTaskTime', 0),
            'login_count': user_activity.get('loginCount', 0),
            'unusual_location': int(user_activity.get('unusualLocation', False))
        }
    
    def detect(self, user_activity):
        features = self.extract_features(user_activity)
        
        # Scale features
        scaled_features = self.scaler.transform_one(features)
        
        # Get anomaly score
        score = self.detector.score_one(scaled_features)
        
        # Update model with new data point
        self.detector.learn_one(scaled_features)
        
        # Threshold for anomaly
        is_anomaly = score > 0.7
        
        return {
            'isAnomaly': is_anomaly,
            'anomalyScore': float(score),
            'reason': self._get_reason(features, is_anomaly)
        }
    
    def _get_reason(self, features, is_anomaly):
        if not is_anomaly:
            return 'Normal activity'
        
        reasons = []
        if features['unusual_location']:
            reasons.append('Login from unusual location')
        if features['login_hour'] < 6 or features['login_hour'] > 22:
            reasons.append('Login at unusual hour')
        if features['tasks_completed'] > 20:
            reasons.append('Unusually high task completion')
        
        return '; '.join(reasons) if reasons else 'Anomalous pattern detected'
```

### FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from models.ticket_classifier import TicketClassifier
from models.anomaly_detector import AnomalyDetector
import os

app = FastAPI(title="Enterprise ML Service")

# Initialize models
ticket_classifier = TicketClassifier()
anomaly_detector = AnomalyDetector()

class TicketRequest(BaseModel):
    text: str

class AnomalyRequest(BaseModel):
    userId: str
    loginTime: int
    loginLocation: str
    tasksCompleted: int = 0
    avgTaskTime: float = 0.0
    loginCount: int = 1
    unusualLocation: bool = False

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    try:
        result = ticket_classifier.predict(request.text)
        
        # Determine priority based on category
        priority_mapping = {
            'technical': 'high',
            'billing': 'high',
            'general': 'medium',
            'feature_request': 'low'
        }
        result['priority'] = priority_mapping.get(result['category'], 'medium')
        
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: AnomalyRequest):
    try:
        user_activity = request.dict()
        result = anomaly_detector.detect(user_activity)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your-secret-key-here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=info
MAX_WORKERS=4
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

### MongoDB Schema Examples

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
});

module.exports = mongoose.model('User', userSchema);
```

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
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
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
  updatedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Time Tracking Implementation

```javascript
// components/TaskTimer.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const TaskTimer = ({ taskId }) => {
  const { token } = useContext(AuthContext);
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const handleStart = () => setIsRunning(true);
  const handlePause = () => setIsRunning(false);
  
  const handleStop = async () => {
    setIsRunning(false);
    
    // Save time to backend
    await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ timeAdded: seconds })
    });
    
    setSeconds(0);
  };

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="task-timer">
      <div className="time-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={handleStart}>Start</button>
        ) : (
          <button onClick={handlePause}>Pause</button>
        )}
        <button onClick={handleStop} disabled={seconds === 0}>Stop & Save</button>
      </div>
    </div>
  );
};

export default TaskTimer;
```

### Protected Route Component

```javascript
// components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) {
    return <div>Loading...</div>;
  }

  if (!user) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <AuthProvider>
