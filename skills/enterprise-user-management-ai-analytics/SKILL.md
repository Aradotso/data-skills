---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management system with task tracking"
  - "implement AI-powered ticket classification"
  - "build admin dashboard with user management"
  - "add AI risk detection to user system"
  - "integrate ML service for burnout detection"
  - "configure user task management with Kanban board"
  - "deploy enterprise user management application"
---

# Enterprise User Management AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system combining React frontend, Node.js backend, and FastAPI ML service. Features include JWT authentication, Kanban task boards, support ticket management, and AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive insights.

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB

### Clone and Install

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
```

Start backend:
```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:
```env
MODEL_PATH=./models
LOG_LEVEL=info
```

Start ML service:
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
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Architecture Overview

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   React     │────▶ │   Node.js   │────▶ │   MongoDB   │
│   Frontend  │      │   Backend   │      │   Database  │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │   FastAPI   │
                     │  ML Service │
                     └─────────────┘
```

## Backend API Reference

### Authentication

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

### User Management

```javascript
// GET /api/users - Get all users (Admin only)
const fetchUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
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
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
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
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
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

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      category: ticketData.category, // 'technical', 'billing', 'general'
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API Reference

### AI Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: 'Support Request'
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', confidence: 0.89 }
  return result;
};
```

### Risk Detection

```javascript
// POST /api/ml/detect-risk
const detectUserRisk = async (userId, behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      loginFrequency: behaviorData.loginFrequency,
      taskCompletionRate: behaviorData.taskCompletionRate,
      failedLogins: behaviorData.failedLogins,
      unusualActivity: behaviorData.unusualActivity
    })
  });
  const result = await response.json();
  // Returns: { riskLevel: 'high', score: 0.78, factors: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userId, workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      tasksCompleted: workloadData.tasksCompleted,
      avgWorkHours: workloadData.avgWorkHours,
      overtimeHours: workloadData.overtimeHours,
      missedDeadlines: workloadData.missedDeadlines,
      weekendWork: workloadData.weekendWork
    })
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 'moderate', score: 0.65, recommendations: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityData.userId,
      timestamp: activityData.timestamp,
      action: activityData.action,
      ipAddress: activityData.ipAddress,
      location: activityData.location,
      deviceInfo: activityData.deviceInfo
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, score: 0.92, reasons: [...] }
  return result;
};
```

### Predictive Insights

```javascript
// POST /api/ml/predict-project-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-project-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      daysRemaining: projectData.daysRemaining,
      teamSize: projectData.teamSize,
      avgTaskTime: projectData.avgTaskTime
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.73, estimatedDelay: 5, factors: [...] }
  return result;
};
```

## Frontend Components

### Admin Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState({});

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch users
    const usersRes = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const usersData = await usersRes.json();
    setUsers(usersData);

    // Fetch analytics
    const analyticsRes = await fetch('http://localhost:5000/api/analytics/overview', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const analyticsData = await analyticsRes.json();
    setAnalytics(analyticsData);
  };

  const handleDeleteUser = async (userId) => {
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/users/${userId}`, {
      method: 'DELETE',
      headers: { 'Authorization': `Bearer ${token}` }
    });
    fetchDashboardData();
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      <div className="analytics-cards">
        <div className="card">
          <h3>Total Users</h3>
          <p>{analytics.totalUsers}</p>
        </div>
        <div className="card">
          <h3>Active Tasks</h3>
          <p>{analytics.activeTasks}</p>
        </div>
        <div className="card">
          <h3>Open Tickets</h3>
          <p>{analytics.openTickets}</p>
        </div>
      </div>
      <div className="users-table">
        <h2>Users</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>
                  <button onClick={() => handleDeleteUser(user._id)}>
                    Delete
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminDashboard;
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
    const token = localStorage.getItem('token');
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
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority ${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status === 'todo' && (
          <button onClick={() => moveTask(task._id, 'in-progress')}>
            Start
          </button>
        )}
        {task.status === 'in-progress' && (
          <button onClick={() => moveTask(task._id, 'done')}>
            Complete
          </button>
        )}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>Done</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const AIAnalytics = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [anomalies, setAnomalies] = useState([]);

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch risk analysis
    const riskRes = await fetch(`http://localhost:5000/api/analytics/risk/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    setRiskData(await riskRes.json());

    // Fetch burnout analysis
    const burnoutRes = await fetch(`http://localhost:5000/api/analytics/burnout/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    setBurnoutData(await burnoutRes.json());

    // Fetch anomalies
    const anomalyRes = await fetch(`http://localhost:5000/api/analytics/anomalies/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    setAnomalies(await anomalyRes.json());
  };

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      {riskData && (
        <div className={`insight-card risk-${riskData.riskLevel}`}>
          <h3>Risk Assessment</h3>
          <p>Level: {riskData.riskLevel}</p>
          <p>Score: {(riskData.score * 100).toFixed(1)}%</p>
          <ul>
            {riskData.factors.map((factor, i) => (
              <li key={i}>{factor}</li>
            ))}
          </ul>
        </div>
      )}

      {burnoutData && (
        <div className={`insight-card burnout-${burnoutData.burnoutRisk}`}>
          <h3>Burnout Detection</h3>
          <p>Risk: {burnoutData.burnoutRisk}</p>
          <p>Score: {(burnoutData.score * 100).toFixed(1)}%</p>
          <h4>Recommendations:</h4>
          <ul>
            {burnoutData.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      {anomalies.length > 0 && (
        <div className="insight-card anomalies">
          <h3>Detected Anomalies</h3>
          {anomalies.map((anomaly, i) => (
            <div key={i} className="anomaly-item">
              <p><strong>{anomaly.action}</strong> at {anomaly.timestamp}</p>
              <p>Reason: {anomaly.reasons.join(', ')}</p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Common Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Protected API Route

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const { authMiddleware, adminOnly } = require('../middleware/auth');

router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

router.put('/:id', authMiddleware, async (req, res) => {
  try {
    // Only admin or self can update
    if (req.user.role !== 'admin' && req.user.id !== req.params.id) {
      return res.status(403).json({ message: 'Unauthorized' });
    }
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### ML Model Integration

```python
# ml-service/models/burnout_detector.py
from river import ensemble, preprocessing
import pickle

class BurnoutDetector:
    def __init__(self):
        self.model = ensemble.AdaptiveRandomForestClassifier(
            n_models=10,
            seed=42
        )
        self.scaler = preprocessing.StandardScaler()
        
    def predict(self, features):
        """
        features: dict with keys:
        - tasks_completed
        - avg_work_hours
        - overtime_hours
        - missed_deadlines
        - weekend_work
        """
        scaled_features = self.scaler.learn_one(features).transform_one(features)
        prediction = self.model.predict_proba_one(scaled_features)
        
        burnout_score = prediction.get(1, 0)
        
        if burnout_score > 0.7:
            risk = "high"
        elif burnout_score > 0.4:
            risk = "moderate"
        else:
            risk = "low"
            
        recommendations = self._get_recommendations(features, burnout_score)
        
        return {
            "burnoutRisk": risk,
            "score": burnout_score,
            "recommendations": recommendations
        }
    
    def _get_recommendations(self, features, score):
        recs = []
        if features['avg_work_hours'] > 45:
            recs.append("Reduce work hours to maintain work-life balance")
        if features['overtime_hours'] > 10:
            recs.append("Limit overtime work")
        if features['weekend_work'] > 2:
            recs.append("Take proper weekends off")
        if features['missed_deadlines'] > 3:
            recs.append("Reassess task allocation and deadlines")
        return recs
```

### FastAPI ML Endpoint

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.burnout_detector import BurnoutDetector
from models.risk_detector import RiskDetector
from models.ticket_classifier import TicketClassifier

app = FastAPI()

burnout_detector = BurnoutDetector()
risk_detector = RiskDetector()
ticket_classifier = TicketClassifier()

class BurnoutRequest(BaseModel):
    userId: str
    tasksCompleted: int
    avgWorkHours: float
    overtimeHours: float
    missedDeadlines: int
    weekendWork: int

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    features = {
        "tasks_completed": request.tasksCompleted,
        "avg_work_hours": request.avgWorkHours,
        "overtime_hours": request.overtimeHours,
        "missed_deadlines": request.missedDeadlines,
        "weekend_work": request.weekendWork
    }
    result = burnout_detector.predict(features)
    return result

class TicketRequest(BaseModel):
    text: str
    subject: str

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    result = ticket_classifier.classify(request.text, request.subject)
    return result
```

## Configuration

### MongoDB Schema Examples

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

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
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('User', userSchema);

// backend/models/Task.js
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: {
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
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);

// backend/models/Ticket.js
const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general'],
    required: true
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
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
    category: String,
    confidence: Number
  },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### Environment Variables Reference

```env
# Backend .env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000

# ML Service .env
MODEL_PATH=./models
LOG_LEVEL=info
RETRAIN_INTERVAL=86400

# Frontend .env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
REACT_APP_ENV=development
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
  } catch (error) {
    return false;
  }
};

// Refresh token if expired
const handleAuthError = (error) => {
  if (error.response?.status === 401) {
    localStorage.removeItem('token');
    window.location.href = '/login';
  }
};
```

### CORS Configuration

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

###
