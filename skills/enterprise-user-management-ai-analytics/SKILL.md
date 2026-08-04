---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly analysis, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "configure user management system with AI features"
  - "implement AI-powered task and ticket management"
  - "add burnout detection and risk analysis to user system"
  - "create admin dashboard with user analytics"
  - "build user management app with ML insights"
  - "integrate AI ticket classification and routing"
  - "deploy full-stack user management with FastAPI ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack application that combines user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication, and user CRUD operations
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Tickets**: Smart ticket routing and classification
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, and predictive insights
- **Admin Dashboard**: Organization-wide analytics and monitoring

The system uses React for the frontend, Node.js/Express for the backend, MongoDB for data storage, and FastAPI with scikit-learn/River for ML services.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or cloud instance)

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

Create `.env` file in `backend/` directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# or for development
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/` directory:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000,http://localhost:5000
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

Create `.env` file in `frontend/` directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Access at `http://localhost:3000`

## Key API Endpoints

### Authentication (Backend)

```javascript
// Register user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
// Returns: { token, user }
```

### User Management (Backend)

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }

// Update user
PUT /api/users/:id
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
{
  "name": "John Smith",
  "role": "manager"
}

// Delete user (Admin only)
DELETE /api/users/:id
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
```

### Task Management (Backend)

```javascript
// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
{
  "title": "Implement feature X",
  "description": "Build new dashboard component",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Get user tasks
GET /api/tasks/user/:userId
```

### Support Tickets (Backend)

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
{
  "subject": "Login issue",
  "description": "Unable to access dashboard after login",
  "priority": "medium",
  "category": "technical"
}

// Get all tickets (Admin)
GET /api/tickets

// Update ticket
PATCH /api/tickets/:id
{
  "status": "resolved",
  "resolution": "Password reset completed"
}
```

### AI/ML Endpoints (ML Service)

```python
# Risk prediction
POST /api/ml/predict-risk
{
  "user_id": "user123",
  "failed_logins": 3,
  "unusual_activity_count": 5,
  "role": "user",
  "account_age_days": 120
}
# Returns: { "risk_score": 0.75, "risk_level": "high" }

# Anomaly detection
POST /api/ml/detect-anomaly
{
  "user_id": "user123",
  "login_time": "2026-04-15T03:30:00",
  "ip_address": "192.168.1.100",
  "location": "New York",
  "device": "mobile"
}
# Returns: { "is_anomaly": true, "anomaly_score": 0.82 }

# Burnout detection
POST /api/ml/detect-burnout
{
  "user_id": "user123",
  "tasks_completed": 45,
  "tasks_pending": 20,
  "avg_task_duration_hours": 6.5,
  "overtime_hours": 15,
  "days_since_break": 30
}
# Returns: { "burnout_risk": "high", "burnout_score": 0.85, "recommendations": [...] }

# Ticket classification
POST /api/ml/classify-ticket
{
  "subject": "Cannot reset password",
  "description": "The password reset link is not working",
  "user_history": []
}
# Returns: { "category": "authentication", "priority": "high", "suggested_assignee": "tech_support" }
```

## Code Examples

### Backend: User Authentication Middleware (Node.js)

```javascript
// middleware/auth.js
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
    req.user = await User.findById(decoded.id);
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

### Backend: Task Controller (Node.js)

```javascript
// controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    
    res.status(201).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message
    });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email');
    
    res.status(200).json({
      success: true,
      count: tasks.length,
      data: tasks
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message
    });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true, runValidators: true }
    );
    
    if (!task) {
      return res.status(404).json({
        success: false,
        error: 'Task not found'
      });
    }
    
    res.status(200).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message
    });
  }
};
```

### ML Service: Risk Prediction (Python/FastAPI)

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("CORS_ORIGINS", "*").split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    unusual_activity_count: int
    role: str
    account_age_days: int

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_task_duration_hours: float
    overtime_hours: float
    days_since_break: int

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Feature engineering
        role_mapping = {"user": 1, "manager": 2, "admin": 3}
        features = np.array([[
            request.failed_logins,
            request.unusual_activity_count,
            role_mapping.get(request.role, 1),
            request.account_age_days
        ]])
        
        # Simple risk calculation (replace with trained model)
        risk_score = min(1.0, (
            request.failed_logins * 0.3 +
            request.unusual_activity_count * 0.2 +
            (1.0 if request.account_age_days < 30 else 0) * 0.5
        ) / 10)
        
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        return {
            "user_id": request.user_id,
            "risk_score": round(risk_score, 2),
            "risk_level": risk_level,
            "factors": {
                "failed_logins": request.failed_logins,
                "unusual_activity": request.unusual_activity_count
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        # Burnout score calculation
        workload_score = min(1.0, request.tasks_pending / 30)
        overtime_score = min(1.0, request.overtime_hours / 20)
        break_score = min(1.0, request.days_since_break / 45)
        
        burnout_score = (workload_score * 0.4 + overtime_score * 0.3 + break_score * 0.3)
        
        risk = "high" if burnout_score > 0.7 else "medium" if burnout_score > 0.4 else "low"
        
        recommendations = []
        if overtime_score > 0.5:
            recommendations.append("Reduce overtime hours")
        if break_score > 0.6:
            recommendations.append("Schedule time off")
        if workload_score > 0.7:
            recommendations.append("Redistribute tasks")
        
        return {
            "user_id": request.user_id,
            "burnout_score": round(burnout_score, 2),
            "burnout_risk": risk,
            "recommendations": recommendations,
            "metrics": {
                "workload_pressure": round(workload_score, 2),
                "overtime_pressure": round(overtime_score, 2),
                "rest_deficit": round(break_score, 2)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

### Frontend: User Dashboard Component (React)

```javascript
// frontend/src/components/UserDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);
  
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;
  
  useEffect(() => {
    fetchUserData();
  }, []);
  
  const fetchUserData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = { headers: { Authorization: `Bearer ${token}` } };
      
      // Fetch tasks
      const tasksRes = await axios.get(`${API_URL}/tasks/my-tasks`, config);
      const tasksByStatus = {
        todo: tasksRes.data.data.filter(t => t.status === 'todo'),
        inProgress: tasksRes.data.data.filter(t => t.status === 'in-progress'),
        done: tasksRes.data.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
      
      // Check burnout risk
      const burnoutRes = await axios.post(`${ML_API_URL}/api/ml/detect-burnout`, {
        user_id: localStorage.getItem('userId'),
        tasks_completed: tasksByStatus.done.length,
        tasks_pending: tasksByStatus.todo.length + tasksByStatus.inProgress.length,
        avg_task_duration_hours: 5.5,
        overtime_hours: 10,
        days_since_break: 20
      });
      setBurnoutData(burnoutRes.data);
      
      setLoading(false);
    } catch (error) {
      console.error('Error fetching user data:', error);
      setLoading(false);
    }
  };
  
  const moveTask = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUserData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutData && burnoutData.burnout_risk === 'high' && (
        <div className="alert alert-warning">
          <strong>Burnout Risk Alert:</strong> {burnoutData.recommendations.join(', ')}
        </div>
      )}
      
      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do ({tasks.todo.length})</h3>
          {tasks.todo.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, 'in-progress')}>
                Start
              </button>
            </div>
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>In Progress ({tasks.inProgress.length})</h3>
          {tasks.inProgress.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, 'done')}>
                Complete
              </button>
            </div>
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>Done ({tasks.done.length})</h3>
          {tasks.done.map(task => (
            <div key={task._id} className="task-card completed">
              <h4>{task.title}</h4>
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

### Frontend: Admin Analytics Component (React)

```javascript
// frontend/src/components/AdminAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);
  
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;
  
  useEffect(() => {
    fetchAnalytics();
  }, []);
  
  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = { headers: { Authorization: `Bearer ${token}` } };
      
      const [usersRes, tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${API_URL}/users`, config),
        axios.get(`${API_URL}/tasks`, config),
        axios.get(`${API_URL}/tickets`, config)
      ]);
      
      // Check risk for each user
      const riskPromises = usersRes.data.data.map(user => 
        axios.post(`${ML_API_URL}/api/ml/predict-risk`, {
          user_id: user._id,
          failed_logins: user.failedLogins || 0,
          unusual_activity_count: user.unusualActivityCount || 0,
          role: user.role,
          account_age_days: Math.floor((Date.now() - new Date(user.createdAt)) / (1000 * 60 * 60 * 24))
        }).catch(() => null)
      );
      
      const riskResults = await Promise.all(riskPromises);
      const highRiskUsers = riskResults
        .filter(r => r && r.data.risk_level === 'high')
        .map(r => r.data);
      
      setRiskUsers(highRiskUsers);
      setAnalytics({
        totalUsers: usersRes.data.data.length,
        totalTasks: tasksRes.data.data.length,
        totalTickets: ticketsRes.data.data.length,
        openTickets: ticketsRes.data.data.filter(t => t.status === 'open').length
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };
  
  if (!analytics) return <div>Loading analytics...</div>;
  
  return (
    <div className="admin-analytics">
      <h1>Organization Analytics</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-number">{analytics.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p className="stat-number">{analytics.totalTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-number">{analytics.openTickets}</p>
        </div>
      </div>
      
      {riskUsers.length > 0 && (
        <div className="risk-alerts">
          <h2>High Risk Users</h2>
          <table>
            <thead>
              <tr>
                <th>User ID</th>
                <th>Risk Score</th>
                <th>Failed Logins</th>
                <th>Action</th>
              </tr>
            </thead>
            <tbody>
              {riskUsers.map(user => (
                <tr key={user.user_id}>
                  <td>{user.user_id}</td>
                  <td>{user.risk_score}</td>
                  <td>{user.factors.failed_logins}</td>
                  <td><button>Investigate</button></td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
};

export default AdminAnalytics;
```

## Configuration

### Database Models (MongoDB/Mongoose)

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please add a name']
  },
  email: {
    type: String,
    required: [true, 'Please add an email'],
    unique: true,
    match: [/^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/, 'Please add a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please add a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'manager', 'admin'],
    default: 'user'
  },
  failedLogins: {
    type: Number,
    default: 0
  },
  unusualActivityCount: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

UserSchema.methods.getSignedJwtToken = function() {
  return jwt.sign({ id: this._id }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRE
  });
};

UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

```javascript
// models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Please add a task title']
  },
  description: {
    type: String,
    required: [true, 'Please add a description']
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
  assignedTo: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', TaskSchema);
```

## Common Patterns

### Integrating ML Predictions in Backend

```javascript
// services/mlService.js
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL;

exports.checkUserRisk = async (user) => {
  try {
    const response = await axios.post(`${ML_SERVICE_URL}/api/ml/predict-risk`, {
      user_id: user._id.toString(),
      failed_logins: user.failedLogins || 0,
      unusual_activity_count: user.unusualActivityCount || 0,
      role: user.role,
      account_age_days: Math.floor((Date.now() - new Date(user.createdAt)) / (1000 * 60 * 60 * 24))
    });
    return response.data;
  } catch (error) {
    console.error('ML service error:', error.message);
    return null;
  }
};

exports.classifyTicket = async (ticket) => {
  try {
    const response = await axios.post(`${ML_SERVICE_URL}/api/ml/classify-ticket`, {
      subject: ticket.subject,
      description: ticket.description,
      user_history: []
    });
    return response.data;
  } catch (error) {
    console.error('ML service error:', error.message);
    return null;
  }
};
```

### Scheduled Risk Checks

```javascript
// jobs/riskChecker.js
const cron = require('node-cron');
const User = require('../models/User');
const mlService = require('../services/mlService');
const sendAlert = require('../utils/alerts');

// Run every day at midnight
cron.schedule('0 0 * * *', async () => {
  console.log('Running daily risk check...');
  
  try {
    const users = await User.find({ role: { $ne: 'admin' } });
    
    for (const user of users) {
      const riskData = await mlService.checkUserRisk(user);
      
      if (riskData && riskData.risk_level === 'high') {
        await sendAlert({
          type: 'high_risk_user',
          userId: user._id,
          riskScore: riskData.risk_score,
          factors: riskData.factors
        });
      }
    }
    
    console.log('Risk check completed');
  } catch (error) {
    console.error('Risk check failed:', error);
  }
});
```

## Troubleshooting

### Issue: JWT Token Expired

```javascript
// Handle token refresh in frontend
const axiosInstance = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

axiosInstance.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### Issue: ML Service Connection Failed

```javascript
// Add fallback when ML service is unavailable
const getRiskPrediction = async (userData) => {
  try {
    const response = await axios.post(
      `${ML_SERVICE_URL}/api/ml/predict-risk`,
      userData,
      { timeout: 5000 }
    );
    return response.data;
  } catch (error) {
    console.warn('ML service unavailable, using default risk assessment');
    // Fallback to rule-based risk
    return {
      risk_score: userData.failed_logins > 5 ? 0.8 : 0.2,
      risk_level: userData.failed_logins > 5 ? 'high' : 'low'
    };
  }
};
```

### Issue: MongoDB Connection Errors

```javascript
// Improved MongoDB connection with retry
const connectDB = async () => {
  const maxRetries = 5;
  let retries = 0;
  
  while (retries < maxRetries) {
    try {
      await mongoose.connect(process.env.MONGODB_URI, {
        useNewUrlParser: true,
        useUnifiedTopology: true
      });
      console.log('MongoDB connected');
      return;
    } catch (error) {
      retries++;
      console.log(`MongoDB connection attempt ${retries} failed:`, error.message);
      await new Promise(resolve => setTimeout(resolve, 5000));
    }
  }
  
  console.error('Could not connect to MongoDB');
  process.exit(1);
};
```

### Issue: CORS Errors

```javascript
// Backend: Proper CORS configuration
const cors = require('cors');

const corsOptions = {
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));
```

### Performance: Batch ML Predictions

```python
# ml-service: Optimize for batch predictions
from typing import List

class BatchRiskRequest(BaseModel):
    users: List[RiskPredictionRequest]

@app.post("/api/ml/predict-risk-batch")
async def predict_risk_batch(request: BatchRiskRequest):
    results = []
    for user in request.users:
        risk_score = min(1.0, (
            user.failed_logins * 0.3 +
            user.unusual_activity_count * 0.2 +
            (1.0 if user.account_age_days < 30 else 0) * 0.5
        ) / 10)
        
