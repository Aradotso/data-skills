---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task management, ticket routing, risk detection, and workforce insights
triggers:
  - "help me set up an enterprise user management system"
  - "how do I implement AI-powered ticket classification"
  - "create a user management dashboard with analytics"
  - "integrate risk detection and anomaly detection for users"
  - "build a task management system with burnout prediction"
  - "implement JWT authentication for user management"
  - "set up AI analytics for enterprise workflows"
  - "configure kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform that combines traditional CRUD operations with AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing. Built with React, Node.js/Express, MongoDB, and FastAPI ML service.

## What This Project Does

This system provides a complete enterprise user management solution with:

- **User Management**: Role-based access control, authentication, user CRUD operations
- **Task Management**: Kanban boards, time tracking, task assignment and monitoring
- **Support Tickets**: AI-powered classification, routing, and status tracking
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, project delay forecasting
- **Admin Dashboard**: Organization-wide analytics, audit logs, alert management

The architecture consists of three main services:
1. Frontend (React) - User interface
2. Backend (Node.js/Express) - API and business logic
3. ML Service (FastAPI) - AI/ML predictions and analytics

## Installation

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start the backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start the frontend:

```bash
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Login
POST /api/auth/login
Body: { "email": "user@example.com", "password": "password123" }

// Register
POST /api/auth/register
Body: { "name": "John Doe", "email": "john@example.com", "password": "pass123", "role": "user" }

// Get current user
GET /api/auth/me
Headers: { "Authorization": "Bearer <token>" }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <admin_token>" }

// Create user
POST /api/users
Body: { "name": "Jane Doe", "email": "jane@example.com", "role": "user", "department": "Engineering" }

// Update user
PUT /api/users/:id
Body: { "role": "admin", "status": "active" }

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/user/:userId

// Create task
POST /api/tasks
Body: { 
  "title": "Implement feature X",
  "description": "Add new authentication flow",
  "assignedTo": "userId",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-12-31"
}

// Update task status
PUT /api/tasks/:id
Body: { "status": "in-progress" }

// Track time
POST /api/tasks/:id/time
Body: { "timeSpent": 3600 }
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Body: {
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when accessing admin panel",
  "priority": "high",
  "category": "technical"
}

// Get user tickets
GET /api/tickets/user/:userId

// Update ticket
PUT /api/tickets/:id
Body: { "status": "resolved", "assignedTo": "adminId" }
```

## AI/ML Endpoints

### Risk Prediction

```javascript
// Predict user risk score
POST /api/ml/predict-risk
Body: {
  "userId": "user123",
  "taskCompletionRate": 0.75,
  "averageTaskTime": 4.5,
  "overdueCount": 2,
  "loginFrequency": 5.2
}

// Response: { "riskScore": 0.35, "riskLevel": "medium", "factors": [...] }
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
POST /api/ml/detect-anomaly
Body: {
  "userId": "user123",
  "loginTime": "2026-04-15T02:30:00Z",
  "ipAddress": "192.168.1.100",
  "activityType": "data_export",
  "resourceAccessed": "sensitive_data"
}

// Response: { "isAnomaly": true, "anomalyScore": 0.87, "reason": "Unusual login time" }
```

### Burnout Detection

```javascript
// Analyze burnout risk
POST /api/ml/burnout-analysis
Body: {
  "userId": "user123",
  "weeklyHours": 55,
  "taskCount": 15,
  "overdueCount": 5,
  "avgResponseTime": 48
}

// Response: { "burnoutScore": 0.72, "level": "high", "recommendations": [...] }
```

### Ticket Classification

```javascript
// Auto-classify and route ticket
POST /api/ml/classify-ticket
Body: {
  "title": "Cannot login to system",
  "description": "Getting authentication error after password reset"
}

// Response: { 
//   "category": "authentication",
//   "priority": "high",
//   "suggestedAssignee": "supportTeamId",
//   "confidence": 0.92
// }
```

## Frontend Integration

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
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
      const response = await axios.get(`${API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/auth/login`, { email, password });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### Task Management Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/tasks/user/${userId}`);
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${API_URL}/tasks/${taskId}`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const onDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const onDrop = (e, newStatus) => {
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');
    if (currentStatus !== newStatus) {
      updateTaskStatus(taskId, newStatus);
    }
  };

  const onDragOver = (e) => e.preventDefault();

  return (
    <div className="kanban-board">
      {Object.keys(tasks).map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => onDrop(e, status)}
          onDragOver={onDragOver}
        >
          <h3>{status.replace('-', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              draggable
              onDragStart={(e) => onDragStart(e, task._id, status)}
              className="task-card"
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>{task.priority}</span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const [risk, burnout, anomalies] = await Promise.all([
        axios.post(`${API_URL}/ml/predict-risk`, { userId }),
        axios.post(`${API_URL}/ml/burnout-analysis`, { userId }),
        axios.get(`${API_URL}/ml/anomalies/${userId}`)
      ]);

      setAnalytics({
        risk: risk.data,
        burnout: burnout.data,
        anomalies: anomalies.data
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics-dashboard">
      <div className="analytics-card risk">
        <h3>Risk Assessment</h3>
        <div className={`score ${analytics.risk.riskLevel}`}>
          {(analytics.risk.riskScore * 100).toFixed(0)}%
        </div>
        <p>Risk Level: {analytics.risk.riskLevel}</p>
        <ul>
          {analytics.risk.factors?.map((factor, i) => (
            <li key={i}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="analytics-card burnout">
        <h3>Burnout Analysis</h3>
        <div className={`score ${analytics.burnout.level}`}>
          {(analytics.burnout.burnoutScore * 100).toFixed(0)}%
        </div>
        <p>Level: {analytics.burnout.level}</p>
        <div className="recommendations">
          <h4>Recommendations:</h4>
          <ul>
            {analytics.burnout.recommendations?.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      </div>

      <div className="analytics-card anomalies">
        <h3>Recent Anomalies</h3>
        {analytics.anomalies.length === 0 ? (
          <p>No anomalies detected</p>
        ) : (
          <ul>
            {analytics.anomalies.map((anomaly, i) => (
              <li key={i} className="anomaly-item">
                <span className="timestamp">{new Date(anomaly.timestamp).toLocaleString()}</span>
                <span className="reason">{anomaly.reason}</span>
                <span className={`score score-${anomaly.severity}`}>
                  {(anomaly.anomalyScore * 100).toFixed(0)}%
                </span>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Backend Implementation

### User Model (MongoDB)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive', 'suspended'], default: 'active' },
  profileImage: String,
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

UserSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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
  dueDate: Date,
  timeTracking: [{
    startTime: Date,
    endTime: Date,
    duration: Number
  }],
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', TaskSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized to access this route` 
      });
    }
    next();
  };
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL;

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      assignedBy: req.user.id
    });

    // Predict task completion time using ML service
    try {
      const prediction = await axios.post(`${ML_SERVICE_URL}/predict-completion`, {
        taskData: task
      });
      task.estimatedCompletion = prediction.data.estimatedDays;
      await task.save();
    } catch (mlError) {
      console.error('ML prediction failed:', mlError);
    }

    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedBy', 'name email')
      .sort('-createdAt');
    res.json(tasks);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { ...req.body, updatedAt: Date.now() },
      { new: true, runValidators: true }
    );

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.timeTracking.push({
      startTime: new Date(Date.now() - timeSpent * 1000),
      endTime: new Date(),
      duration: timeSpent
    });

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from river import linear_model, metrics
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

MODEL_PATH = os.getenv('MODEL_PATH', './models')

# Load or initialize models
try:
    risk_model = joblib.load(f'{MODEL_PATH}/risk_model.pkl')
except:
    risk_model = RandomForestClassifier(n_estimators=100)

try:
    anomaly_detector = joblib.load(f'{MODEL_PATH}/anomaly_model.pkl')
except:
    anomaly_detector = IsolationForest(contamination=0.1)

burnout_model = linear_model.LogisticRegression()

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCompletionRate: float
    averageTaskTime: float
    overdueCount: int
    loginFrequency: float

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    ipAddress: str
    activityType: str
    resourceAccessed: str

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    weeklyHours: float
    taskCount: int
    overdueCount: int
    avgResponseTime: float

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        features = np.array([[
            request.taskCompletionRate,
            request.averageTaskTime,
            request.overdueCount,
            request.loginFrequency
        ]])
        
        # Simple rule-based risk scoring if model not trained
        risk_score = 0.0
        factors = []
        
        if request.taskCompletionRate < 0.7:
            risk_score += 0.3
            factors.append("Low task completion rate")
        
        if request.overdueCount > 3:
            risk_score += 0.25
            factors.append("High number of overdue tasks")
        
        if request.loginFrequency < 3:
            risk_score += 0.2
            factors.append("Low engagement")
        
        if request.averageTaskTime > 8:
            risk_score += 0.25
            factors.append("Extended task completion times")
        
        risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.6 else "high"
        
        return {
            "riskScore": min(risk_score, 1.0),
            "riskLevel": risk_level,
            "factors": factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        from datetime import datetime
        
        # Parse login time
        login_hour = datetime.fromisoformat(request.loginTime.replace('Z', '+00:00')).hour
        
        # Rule-based anomaly detection
        is_anomaly = False
        anomaly_score = 0.0
        reason = ""
        
        # Check for unusual login times (2 AM - 6 AM)
        if 2 <= login_hour <= 6:
            is_anomaly = True
            anomaly_score += 0.4
            reason = "Unusual login time"
        
        # Check for sensitive data access
        if request.activityType == "data_export" and "sensitive" in request.resourceAccessed.lower():
            is_anomaly = True
            anomaly_score += 0.5
            reason += ("; " if reason else "") + "Sensitive data access"
        
        return {
            "isAnomaly": is_anomaly,
            "anomalyScore": min(anomaly_score, 1.0),
            "reason": reason or "No anomaly detected",
            "timestamp": request.loginTime
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/burnout-analysis")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        burnout_score = 0.0
        recommendations = []
        
        # Weekly hours check
        if request.weeklyHours > 50:
            burnout_score += 0.3
            recommendations.append("Reduce weekly working hours")
        
        # Task load check
        if request.taskCount > 12:
            burnout_score += 0.25
            recommendations.append("Delegate some tasks")
        
        # Overdue tasks check
        if request.overdueCount > 4:
            burnout_score += 0.25
            recommendations.append("Prioritize and reschedule overdue tasks")
        
        # Response time check (in hours)
        if request.avgResponseTime > 36:
            burnout_score += 0.2
            recommendations.append("Improve task engagement and responsiveness")
        
        level = "low" if burnout_score < 0.3 else "medium" if burnout_score < 0.6 else "high"
        
        if burnout_score > 0.5:
            recommendations.append("Consider taking time off or vacation")
        
        return {
            "burnoutScore": min(burnout_score, 1.0),
            "level": level,
            "recommendations": recommendations if recommendations else ["Maintain current workload balance"]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        text = (request.title + " " + request.description).lower()
        
        # Simple keyword-based classification
        if any(word in text for word in ["login", "password", "authentication", "access denied"]):
            category = "authentication"
            priority = "high"
        elif any(word in text for word in ["bug", "error", "crash", "not working"]):
            category = "technical"
            priority = "high"
        elif any(word in text for word in ["feature", "request", "add", "improve"]):
            category = "feature_request"
            priority = "medium"
        elif any(word in text for word in ["question", "how to", "help"]):
            category = "support"
            priority = "low"
        else:
            category = "general"
            priority = "medium"
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAssignee": f"{category}_team",
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Common Patterns

### Protected Route Pattern

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const taskController = require('../controllers/taskController');

// All routes require authentication
router.use(protect);

router.get('/user/:userId', taskController.getUserTasks);
router.post('/', authorize('admin'), taskController.createTask);
router.put('/:id', taskController.updateTaskStatus);
router.post('/:id/time', taskController.trackTime);

module.exports = router;
```

### ML Service Integration Pattern

```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_URL = process.env.ML_SERVICE_URL;

class MLService {
  async predictRisk(userData) {
    try {
      const response = await axios.post(`${ML_URL}/predict-risk`, userData);
      return response.data;
    } catch (error) {
      console.error('ML Service Error:', error.message);
      return null;
    }
  }

  async detectAnomaly(activityData) {
    try {
      const response = await axios.post(`${ML_URL}/detect-anomaly`, activityData);
      return response.data;
    } catch (error) {
      console.error('ML Service Error:', error.message);
      return null;
    }
  }

  async analyzeBurnout(workloadData) {
    try {
      const response = await axios.post(`${ML_URL}/burnout-analysis`, workloadData);
      return response.data;
    } catch (error) {
      console.error('ML Service Error:', error.message);
      return null;
    }
  }

  async classifyTicket(ticketData) {
    try {
      const response = await axios.post(`${ML_URL}/classify-ticket`, ticketData);
      return response.data;
    } catch (error) {
      console.error('ML Service Error:', error.message);
      return null;
    }
  }
}

module.exports = new MLService();
```

## Configuration

### Backend Configuration

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

### Server Setup

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');
const dotenv = require('dotenv');
const connectDB = require('./config/database');

dotenv.config();

// Connect to database
connectDB();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/ml', require('./routes/ml'));

// Error handler
app.use((
