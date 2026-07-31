---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management built with React, Node.js, and FastAPI.
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "show me how to implement task tracking with AI insights"
  - "create a user management dashboard with risk detection"
  - "implement ticket classification with machine learning"
  - "build an admin panel with anomaly detection"
  - "add burnout analysis to my user management app"
  - "integrate JWT authentication with role-based access control"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System with AI Analytics - a comprehensive full-stack application that combines user management, task tracking, support ticketing, and AI-powered insights for enterprise organizations.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Centralized user administration with role-based access control (RBAC)
- **Task Management**: Kanban-style task boards with time tracking
- **Support Ticketing**: AI-powered ticket classification and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, and predictive insights
- **Authentication**: JWT-based secure authentication
- **Admin Dashboard**: Comprehensive analytics and audit logging

## Architecture Overview

The system consists of three main components:

1. **Frontend** (React.js): User interface for admins and users
2. **Backend** (Node.js): REST API server handling business logic
3. **ML Service** (FastAPI + scikit-learn): AI/ML microservice for analytics

## Installation

### Prerequisites

```bash
node >= 14.x
npm >= 6.x
python >= 3.8
pip >= 20.x
mongodb >= 4.x
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

Create `.env` file:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Development mode with auto-reload
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MONGO_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# Production mode
uvicorn main:app --host 0.0.0.0 --port 8000
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

Start frontend:

```bash
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Login
POST /api/auth/login
Content-Type: application/json

{
  "email": "admin@example.com",
  "password": "password123"
}

// Response
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "123",
    "name": "Admin User",
    "email": "admin@example.com",
    "role": "admin"
  }
}
```

### User Management

```javascript
// Get all users (Admin only)
GET /api/users
Authorization: Bearer <token>

// Create user
POST /api/users
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:id
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "John Updated",
  "role": "manager"
}

// Delete user
DELETE /api/users/:id
Authorization: Bearer <token>
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/user/:userId
Authorization: Bearer <token>

// Create task
POST /api/tasks
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "Implement authentication",
  "description": "Add JWT-based authentication",
  "assignedTo": "userId123",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:id/status
Authorization: Bearer <token>
Content-Type: application/json

{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:id/time-log
Authorization: Bearer <token>
Content-Type: application/json

{
  "duration": 3600, // seconds
  "date": "2026-04-15"
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "Login issue",
  "description": "Cannot login with valid credentials",
  "priority": "high",
  "category": "technical"
}

// Get tickets
GET /api/tickets
Authorization: Bearer <token>

// Update ticket status
PATCH /api/tickets/:id
Authorization: Bearer <token>
Content-Type: application/json

{
  "status": "resolved",
  "resolution": "Password reset completed"
}
```

## ML Service API

### AI Analytics Endpoints

```python
# Risk Detection
POST /api/ml/risk-detection
Content-Type: application/json

{
  "userId": "user123",
  "taskCompletionRate": 0.65,
  "avgTaskDelay": 2.5,
  "failedLoginAttempts": 3,
  "accessPatternScore": 0.8
}

# Response
{
  "riskLevel": "medium",
  "riskScore": 0.62,
  "factors": ["low_completion_rate", "task_delays"],
  "recommendations": ["Review workload", "Provide support"]
}
```

```python
# Anomaly Detection
POST /api/ml/anomaly-detection
Content-Type: application/json

{
  "userId": "user123",
  "loginTime": "03:00:00",
  "loginLocation": "Unknown Location",
  "accessPattern": "unusual",
  "dataVolume": 10000
}

# Response
{
  "isAnomaly": true,
  "anomalyScore": 0.85,
  "alertLevel": "high",
  "detectedAnomalies": ["unusual_time", "unknown_location"]
}
```

```python
# Burnout Detection
POST /api/ml/burnout-detection
Content-Type: application/json

{
  "userId": "user123",
  "avgWorkHours": 55,
  "taskLoad": 25,
  "missedDeadlines": 5,
  "lastBreakDays": 45
}

# Response
{
  "burnoutRisk": "high",
  "burnoutScore": 0.78,
  "factors": ["excessive_hours", "high_task_load", "no_breaks"],
  "recommendations": ["Reduce workload", "Schedule vacation"]
}
```

```python
# Ticket Classification
POST /api/ml/classify-ticket
Content-Type: application/json

{
  "title": "Cannot access database",
  "description": "Getting connection timeout errors when trying to connect to production DB",
  "priority": "high"
}

# Response
{
  "category": "technical",
  "suggestedAssignee": "database_admin",
  "estimatedResolutionTime": "4 hours",
  "confidence": 0.92
}
```

## Code Examples

### Backend - User Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

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

### Backend - Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');

const createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user.id,
      status: 'todo'
    });

    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

const updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;

    const task = await Task.findById(id);
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.status = status;
    task.statusHistory.push({
      status,
      changedAt: new Date(),
      changedBy: req.user.id
    });

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

const getUserTasks = async (req, res) => {
  try {
    const { userId } = req.params;
    const tasks = await Task.find({ assignedTo: userId })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name')
      .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

module.exports = { createTask, updateTaskStatus, getUserTasks };
```

### ML Service - Risk Detection Model

```python
# ml_service/models/risk_detector.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskDetector:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.model = self._load_or_create_model()
        
    def _load_or_create_model(self):
        if os.path.exists(self.model_path):
            return joblib.load(self.model_path)
        else:
            # Create new model
            model = RandomForestClassifier(n_estimators=100, random_state=42)
            return model
    
    def predict_risk(self, features):
        """
        features: dict with keys:
        - taskCompletionRate
        - avgTaskDelay
        - failedLoginAttempts
        - accessPatternScore
        """
        X = np.array([[
            features['taskCompletionRate'],
            features['avgTaskDelay'],
            features['failedLoginAttempts'],
            features['accessPatternScore']
        ]])
        
        if hasattr(self.model, 'predict_proba'):
            risk_score = self.model.predict_proba(X)[0][1]
        else:
            # Fallback to rule-based
            risk_score = self._rule_based_risk(features)
        
        risk_level = self._get_risk_level(risk_score)
        factors = self._identify_risk_factors(features)
        
        return {
            'riskLevel': risk_level,
            'riskScore': float(risk_score),
            'factors': factors,
            'recommendations': self._get_recommendations(factors)
        }
    
    def _rule_based_risk(self, features):
        score = 0.0
        if features['taskCompletionRate'] < 0.7:
            score += 0.3
        if features['avgTaskDelay'] > 2:
            score += 0.25
        if features['failedLoginAttempts'] > 3:
            score += 0.25
        if features['accessPatternScore'] < 0.5:
            score += 0.2
        return min(score, 1.0)
    
    def _get_risk_level(self, score):
        if score < 0.3:
            return 'low'
        elif score < 0.7:
            return 'medium'
        else:
            return 'high'
    
    def _identify_risk_factors(self, features):
        factors = []
        if features['taskCompletionRate'] < 0.7:
            factors.append('low_completion_rate')
        if features['avgTaskDelay'] > 2:
            factors.append('task_delays')
        if features['failedLoginAttempts'] > 3:
            factors.append('failed_logins')
        if features['accessPatternScore'] < 0.5:
            factors.append('unusual_access_pattern')
        return factors
    
    def _get_recommendations(self, factors):
        recommendations = []
        if 'low_completion_rate' in factors:
            recommendations.append('Review workload distribution')
        if 'task_delays' in factors:
            recommendations.append('Provide additional support or training')
        if 'failed_logins' in factors:
            recommendations.append('Verify account security')
        if 'unusual_access_pattern' in factors:
            recommendations.append('Monitor user activity')
        return recommendations
    
    def train(self, X, y):
        self.model.fit(X, y)
        joblib.dump(self.model, self.model_path)
```

### ML Service - FastAPI Endpoints

```python
# ml_service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_detector import RiskDetector
from models.anomaly_detector import AnomalyDetector
from models.burnout_detector import BurnoutDetector
from models.ticket_classifier import TicketClassifier

app = FastAPI(title="Enterprise AI Analytics Service")

# Initialize models
risk_detector = RiskDetector()
anomaly_detector = AnomalyDetector()
burnout_detector = BurnoutDetector()
ticket_classifier = TicketClassifier()

class RiskRequest(BaseModel):
    userId: str
    taskCompletionRate: float
    avgTaskDelay: float
    failedLoginAttempts: int
    accessPatternScore: float

class AnomalyRequest(BaseModel):
    userId: str
    loginTime: str
    loginLocation: str
    accessPattern: str
    dataVolume: int

class BurnoutRequest(BaseModel):
    userId: str
    avgWorkHours: float
    taskLoad: int
    missedDeadlines: int
    lastBreakDays: int

class TicketRequest(BaseModel):
    title: str
    description: str
    priority: str

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskRequest):
    try:
        result = risk_detector.predict_risk(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(request: AnomalyRequest):
    try:
        result = anomaly_detector.detect(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutRequest):
    try:
        result = burnout_detector.predict(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    try:
        result = ticket_classifier.classify(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

### Frontend - React Task Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks/user/${userId}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );

      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };

      setTasks(categorized);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority ${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => updateTaskStatus(
            task._id, 
            task.status === 'todo' ? 'in-progress' : 'done'
          )}>
            {task.status === 'todo' ? 'Start' : 'Complete'}
          </button>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="task-board">
      <div className="column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### Frontend - AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    risk: null,
    burnout: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      
      // Fetch user metrics
      const metricsResponse = await axios.get(
        `${process.env.REACT_APP_API_URL}/users/${userId}/metrics`,
        { headers: { Authorization: `Bearer ${token}` } }
      );

      const metrics = metricsResponse.data;

      // Get risk analysis
      const riskResponse = await axios.post(
        `${process.env.REACT_APP_ML_URL}/api/ml/risk-detection`,
        {
          userId,
          taskCompletionRate: metrics.completionRate,
          avgTaskDelay: metrics.avgDelay,
          failedLoginAttempts: metrics.failedLogins,
          accessPatternScore: metrics.accessScore
        }
      );

      // Get burnout analysis
      const burnoutResponse = await axios.post(
        `${process.env.REACT_APP_ML_URL}/api/ml/burnout-detection`,
        {
          userId,
          avgWorkHours: metrics.avgWorkHours,
          taskLoad: metrics.activeTasks,
          missedDeadlines: metrics.missedDeadlines,
          lastBreakDays: metrics.daysSinceBreak
        }
      );

      setAnalytics({
        risk: riskResponse.data,
        burnout: burnoutResponse.data,
        anomalies: [] // Would fetch from anomaly detection endpoint
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const getRiskColor = (level) => {
    switch(level) {
      case 'high': return '#ff4444';
      case 'medium': return '#ffbb33';
      case 'low': return '#00C851';
      default: return '#666';
    }
  };

  return (
    <div className="ai-analytics">
      <h2>AI Analytics Dashboard</h2>
      
      {analytics.risk && (
        <div className="analytics-card">
          <h3>Risk Assessment</h3>
          <div 
            className="risk-indicator" 
            style={{ backgroundColor: getRiskColor(analytics.risk.riskLevel) }}
          >
            {analytics.risk.riskLevel.toUpperCase()}
          </div>
          <p>Score: {(analytics.risk.riskScore * 100).toFixed(0)}%</p>
          
          <h4>Risk Factors:</h4>
          <ul>
            {analytics.risk.factors.map((factor, idx) => (
              <li key={idx}>{factor.replace(/_/g, ' ')}</li>
            ))}
          </ul>
          
          <h4>Recommendations:</h4>
          <ul>
            {analytics.risk.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      {analytics.burnout && (
        <div className="analytics-card">
          <h3>Burnout Risk</h3>
          <div 
            className="burnout-indicator"
            style={{ backgroundColor: getRiskColor(analytics.burnout.burnoutRisk) }}
          >
            {analytics.burnout.burnoutRisk.toUpperCase()}
          </div>
          <p>Score: {(analytics.burnout.burnoutScore * 100).toFixed(0)}%</p>
          
          <h4>Contributing Factors:</h4>
          <ul>
            {analytics.burnout.factors.map((factor, idx) => (
              <li key={idx}>{factor.replace(/_/g, ' ')}</li>
            ))}
          </ul>
          
          <h4>Recommendations:</h4>
          <ul>
            {analytics.burnout.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Common Patterns

### Role-Based Access Control

```javascript
// backend/middleware/rbac.js
const checkPermission = (requiredRole) => {
  return (req, res, next) => {
    const userRole = req.user.role;
    const roleHierarchy = { user: 1, manager: 2, admin: 3 };
    
    if (roleHierarchy[userRole] >= roleHierarchy[requiredRole]) {
      next();
    } else {
      res.status(403).json({ error: 'Insufficient permissions' });
    }
  };
};

// Usage in routes
router.post('/users', authMiddleware, checkPermission('admin'), createUser);
router.get('/users/:id', authMiddleware, checkPermission('manager'), getUser);
```

### Time Tracking Implementation

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
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

  const saveTimeLog = async () => {
    const token = localStorage.getItem('token');
    await axios.post(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/time-log`,
      { duration: seconds, date: new Date().toISOString() },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setSeconds(0);
  };

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const seconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={saveTimeLog} disabled={seconds === 0}>
        Save
      </button>
    </div>
  );
};
```

## Configuration

### MongoDB Schema Examples

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { 
    type: String, 
    enum: ['user', 'manager', 'admin'], 
    default: 'user' 
  },
  department: String,
  isActive: { type: Boolean, default: true },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  failedLoginAttempts: { type: Number, default: 0 }
});

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
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
  timeLogs: [{
    duration: Number,
    date: Date,
    loggedAt: { type: Date, default: Date.now }
  }],
  statusHistory: [{
    status: String,
    changedAt: Date,
    changedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' }
  }],
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Troubleshooting

### Common Issues

**Issue: JWT token expires too quickly**
```javascript
// Adjust JWT_EXPIRE in .env
JWT_EXPIRE=7d  // or 30d for longer sessions

// Implement token refresh
router.post('/auth/refresh', async (req, res) => {
  const { refreshToken } = req.body;
  // Verify and issue new token
});
```

**Issue: ML service not connecting to backend**
```bash
# Check ML service is running
curl http://localhost:8000/health

# Verify ML_SERVICE_URL in backend .env
ML_SERVICE_URL=http://localhost:8000

# Check CORS settings in FastAPI
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Issue: MongoDB connection errors**
```javascript
// Add connection error handling
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('MongoDB connected'))
.catch(err => {
  console.error('MongoDB connection error:', err);
  process.exit(1);
});
```

**Issue: React app not loading environment variables**
```bash
# Ensure variables start with REACT_APP_
REACT_APP_API_URL=http://localhost:5000/api

# Restart development server after changing .env
npm start
```

**Issue: CORS errors in production**
