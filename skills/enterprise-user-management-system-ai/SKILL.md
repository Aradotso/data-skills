---
name: enterprise-user-management-system-ai
description: Full-stack user management system with AI analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "build a user management system with AI features"
  - "create admin dashboard with task and ticket management"
  - "implement AI-powered user analytics and risk detection"
  - "add Kanban board with time tracking for users"
  - "integrate ML service for burnout and anomaly detection"
  - "deploy user management system with JWT authentication"
  - "configure AI assistant for ticket classification"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack JavaScript application that provides centralized user, task, and ticket management with built-in AI capabilities. The system features role-based access control, Kanban boards, time tracking, and ML-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

**Architecture:**
- **Frontend**: React.js with component-based UI
- **Backend**: Node.js/Express REST API
- **ML Service**: FastAPI with scikit-learn and River for online learning
- **Database**: MongoDB
- **Authentication**: JWT-based secure authentication

## Installation

### Prerequisites

```bash
# Ensure you have installed:
node --version  # v14+ required
python --version  # 3.8+ required
npm --version
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
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend server
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Frontend runs at http://localhost:3000
```

## Key Components and APIs

### Backend API Endpoints

#### Authentication
```javascript
// POST /api/auth/register
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
  return response.json();
};

// POST /api/auth/login
// Authenticate user
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

#### User Management (Admin)
```javascript
// GET /api/users
// Fetch all users (admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id
// Update user
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
// Delete user
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

#### Task Management
```javascript
// POST /api/tasks
// Create task
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
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'inprogress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId
// Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status
// Update task status (Kanban)
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'todo', 'inprogress', 'done'
  });
  return response.json();
};
```

#### Ticket Management
```javascript
// POST /api/tickets
// Create support ticket
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

// GET /api/tickets
// Get all tickets (admin)
const getAllTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### ML Service API Endpoints

#### AI Ticket Classification
```javascript
// POST /classify-ticket
// Classify ticket using AI
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      title: "System performance issue"
    })
  });
  return response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.87 }
};
```

#### Risk Detection
```javascript
// POST /detect-risk
// Analyze user behavior for risks
const detectUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/detect-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.id,
      loginAttempts: userData.loginAttempts,
      failedLogins: userData.failedLogins,
      lastLoginTime: userData.lastLoginTime,
      activityPattern: userData.activityPattern
    })
  });
  return response.json();
  // Returns: { riskScore: 0.72, riskLevel: 'medium', factors: [...] }
};
```

#### Anomaly Detection
```javascript
// POST /detect-anomaly
// Detect anomalous behavior
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      timestamp: activityData.timestamp,
      action: activityData.action,
      duration: activityData.duration,
      resourceAccessed: activityData.resourceAccessed
    })
  });
  return response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.91, reason: 'Unusual access time' }
};
```

#### Burnout Detection
```javascript
// POST /detect-burnout
// Analyze workload for burnout risk
const detectBurnout = async (workloadData) => {
  const response = await fetch('http://localhost:8000/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: workloadData.userId,
      tasksCompleted: workloadData.tasksCompleted,
      averageWorkHours: workloadData.averageWorkHours,
      overtimeHours: workloadData.overtimeHours,
      weekendWork: workloadData.weekendWork,
      missedDeadlines: workloadData.missedDeadlines
    })
  });
  return response.json();
  // Returns: { burnoutRisk: 'high', score: 0.78, recommendations: [...] }
};
```

#### Predictive Insights
```javascript
// POST /predict-project-delay
// Predict project delay probability
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict-project-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.id,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      daysRemaining: projectData.daysRemaining,
      teamSize: projectData.teamSize,
      complexity: projectData.complexity
    })
  });
  return response.json();
  // Returns: { delayProbability: 0.65, estimatedDelay: 5, factors: [...] }
};
```

## React Component Patterns

### User Dashboard Component
```javascript
// src/components/UserDashboard.js
import React, { useState, useEffect } from 'react';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [user, setUser] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUserData();
    fetchTasks();
  }, []);

  const fetchUserData = async () => {
    const response = await fetch('http://localhost:5000/api/auth/me', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setUser(data);
  };

  const fetchTasks = async () => {
    const response = await fetch(`http://localhost:5000/api/tasks/my-tasks`, {
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

  return (
    <div className="user-dashboard">
      <h1>Welcome, {user?.username}</h1>
      
      <div className="kanban-board">
        <div className="column">
          <h2>To Do</h2>
          {tasks.todo.map(task => (
            <div key={task._id} className="task-card">
              <h3>{task.title}</h3>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, 'inprogress')}>
                Start Task
              </button>
            </div>
          ))}
        </div>

        <div className="column">
          <h2>In Progress</h2>
          {tasks.inprogress.map(task => (
            <div key={task._id} className="task-card active">
              <h3>{task.title}</h3>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, 'done')}>
                Complete
              </button>
            </div>
          ))}
        </div>

        <div className="column">
          <h2>Done</h2>
          {tasks.done.map(task => (
            <div key={task._id} className="task-card completed">
              <h3>{task.title}</h3>
              <p>{task.description}</p>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard
```javascript
// src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [risks, setRisks] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchAnalytics();
    fetchRiskAlerts();
  }, []);

  const fetchAnalytics = async () => {
    const response = await fetch('http://localhost:5000/api/admin/analytics', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setAnalytics(data);
  };

  const fetchRiskAlerts = async () => {
    const response = await fetch('http://localhost:5000/api/admin/risk-alerts', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setRisks(data);
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Analytics Dashboard</h1>
      
      {analytics && (
        <div className="metrics">
          <div className="metric-card">
            <h3>Total Users</h3>
            <p className="metric-value">{analytics.totalUsers}</p>
          </div>
          <div className="metric-card">
            <h3>Active Tasks</h3>
            <p className="metric-value">{analytics.activeTasks}</p>
          </div>
          <div className="metric-card">
            <h3>Open Tickets</h3>
            <p className="metric-value">{analytics.openTickets}</p>
          </div>
          <div className="metric-card">
            <h3>Completion Rate</h3>
            <p className="metric-value">{analytics.completionRate}%</p>
          </div>
        </div>
      )}

      <div className="risk-alerts">
        <h2>Risk Alerts</h2>
        {risks.map(risk => (
          <div key={risk.id} className={`alert alert-${risk.level}`}>
            <strong>{risk.user}</strong>: {risk.message}
            <span className="risk-score">Risk: {risk.score}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

### Time Tracking Component
```javascript
// src/components/TimeTracker.js
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isActive, setIsActive] = useState(false);
  const token = localStorage.getItem('token');

  useEffect(() => {
    let interval = null;
    if (isActive) {
      interval = setInterval(() => {
        setSeconds(seconds => seconds + 1);
      }, 1000);
    } else if (!isActive && seconds !== 0) {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isActive, seconds]);

  const toggle = () => {
    setIsActive(!isActive);
  };

  const reset = () => {
    setSeconds(0);
    setIsActive(false);
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
    reset();
  };

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        <button onClick={toggle}>
          {isActive ? 'Pause' : 'Start'}
        </button>
        <button onClick={reset}>Reset</button>
        <button onClick={saveTime}>Save</button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

## Backend Implementation Patterns

### Express Server Setup
```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const jwt = require('jsonwebtoken');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// MongoDB connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('MongoDB connected'))
.catch(err => console.error('MongoDB error:', err));

// JWT middleware
const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  
  if (!token) return res.sendStatus(401);
  
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) return res.sendStatus(403);
    req.user = user;
    next();
  });
};

// Admin check middleware
const isAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

// Routes
const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/users');
const taskRoutes = require('./routes/tasks');
const ticketRoutes = require('./routes/tickets');

app.use('/api/auth', authRoutes);
app.use('/api/users', authenticateToken, userRoutes);
app.use('/api/tasks', authenticateToken, taskRoutes);
app.use('/api/tickets', authenticateToken, ticketRoutes);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### MongoDB Models
```javascript
// backend/models/User.js
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
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date,
  failedLoginAttempts: {
    type: Number,
    default: 0
  }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(password) {
  return bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
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
    enum: ['todo', 'inprogress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/models/Ticket.js
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
  category: {
    type: String,
    enum: ['technical', 'administrative', 'general'],
    default: 'general'
  },
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  aiClassification: {
    category: String,
    confidence: Number
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

### FastAPI Service Structure
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

app = FastAPI()

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
class TicketClassification(BaseModel):
    text: str
    title: str

class RiskDetection(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    lastLoginTime: str
    activityPattern: List[float]

class BurnoutAnalysis(BaseModel):
    userId: str
    tasksCompleted: int
    averageWorkHours: float
    overtimeHours: float
    weekendWork: int
    missedDeadlines: int

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Ticket classifier
try:
    ticket_vectorizer = joblib.load(f'{MODEL_PATH}/ticket_vectorizer.pkl')
    ticket_classifier = joblib.load(f'{MODEL_PATH}/ticket_classifier.pkl')
except:
    ticket_vectorizer = TfidfVectorizer(max_features=1000)
    ticket_classifier = MultinomialNB()

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketClassification):
    """Classify ticket category and priority using AI"""
    try:
        combined_text = f"{ticket.title} {ticket.text}"
        
        # Simple rule-based classification (can be replaced with trained model)
        text_lower = combined_text.lower()
        
        # Category classification
        if any(word in text_lower for word in ['bug', 'error', 'crash', 'performance']):
            category = 'technical'
            priority = 'high'
        elif any(word in text_lower for word in ['access', 'permission', 'account']):
            category = 'administrative'
            priority = 'medium'
        else:
            category = 'general'
            priority = 'low'
        
        # Priority boost for urgent keywords
        if any(word in text_lower for word in ['urgent', 'critical', 'asap', 'emergency']):
            priority = 'critical'
        
        confidence = np.random.uniform(0.7, 0.95)  # Simulated confidence
        
        return {
            "category": category,
            "priority": priority,
            "confidence": round(confidence, 2)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-risk")
async def detect_risk(data: RiskDetection):
    """Detect user risk based on behavior patterns"""
    try:
        # Calculate risk score
        risk_factors = []
        risk_score = 0.0
        
        # Failed login attempts
        if data.failedLogins > 3:
            risk_score += 0.3
            risk_factors.append("Multiple failed login attempts")
        
        # Login frequency
        if data.loginAttempts > 50:
            risk_score += 0.2
            risk_factors.append("Unusual login frequency")
        
        # Activity pattern anomalies
        if len(data.activityPattern) > 0:
            avg_activity = np.mean(data.activityPattern)
            std_activity = np.std(data.activityPattern)
            if std_activity > 0.5:
                risk_score += 0.25
                risk_factors.append("Irregular activity pattern")
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = 'high'
        elif risk_score >= 0.4:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        return {
            "riskScore": round(risk_score, 2),
            "riskLevel": risk_level,
            "factors": risk_factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(activity: dict):
    """Detect anomalous user behavior"""
    try:
        is_anomaly = False
        anomaly_score = 0.0
        reasons = []
        
        # Check for unusual access times (e.g., 2-5 AM)
        if 'timestamp' in activity:
            hour = int(activity['timestamp'].split(':')[0])
            if 2 <= hour <= 5:
                is_anomaly = True
                anomaly_score = 0.8
                reasons.append("Unusual access time")
        
        # Check action frequency
        if activity.get('duration', 0) > 3600:  # More than 1 hour
            anomaly_score += 0.3
            reasons.append("Unusually long session")
        
        return {
            "isAnomaly": is_anomaly or anomaly_score > 0.5,
            "anomalyScore": round(min(anomaly_score, 1.0), 2),
            "reason": ", ".join(reasons) if reasons else "Normal behavior"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(data: BurnoutAnalysis):
    """Analyze workload for burnout risk"""
    try:
        burnout_score = 0.0
        recommendations = []
        
        # Overtime analysis
        if data.overtimeHours > 10:
            burnout_score += 0.3
            recommendations.append("Reduce overtime hours")
        
        # Weekend work
        if data.weekendWork > 2:
            burnout_score += 0.25
            recommendations.append("Limit weekend work")
        
        # Missed deadlines
        if data.missedDeadlines > 3:
            burnout_score += 0
