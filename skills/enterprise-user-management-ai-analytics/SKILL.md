---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket classification, and risk detection
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics for user behavior monitoring
  - implement JWT authentication with user roles
  - build a kanban board for task management
  - add AI ticket classification and routing
  - create admin dashboard with analytics
  - detect user anomalies and burnout patterns
  - set up ML service for predictive insights
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with a full-stack enterprise user management system that combines React frontend, Node.js backend, and FastAPI ML service for AI-powered analytics including risk detection, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based authentication (Admin/User) with JWT
- **Task Management**: Kanban boards with time tracking and progress monitoring
- **Support Tickets**: AI-powered classification and automatic routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project insights
- **Admin Dashboard**: Organization-wide analytics, audit logs, and user monitoring

## Architecture

The system consists of three main components:
1. **Frontend** (React.js): User interface on port 3000
2. **Backend** (Node.js): REST API and business logic on port 5000
3. **ML Service** (FastAPI): AI/ML models on port 8000

## Installation & Setup

### Prerequisites
```bash
# Required software
node --version  # v14+
python --version  # 3.8+
mongod --version  # MongoDB 4.4+
```

### Quick Start

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install and run backend
cd backend
npm install
npm start  # Runs on http://localhost:5000

# Install and run ML service (new terminal)
cd ml-service
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000

# Install and run frontend (new terminal)
cd frontend
npm install
npm start  # Runs on http://localhost:3000
```

## Backend Configuration

### Environment Variables

Create `backend/.env`:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

### Key Backend API Endpoints

```javascript
// Authentication
POST /api/auth/register
POST /api/auth/login
GET /api/auth/me

// User Management (Admin)
GET /api/users
POST /api/users
PUT /api/users/:id
DELETE /api/users/:id

// Task Management
GET /api/tasks
POST /api/tasks
PUT /api/tasks/:id
DELETE /api/tasks/:id
PATCH /api/tasks/:id/status

// Support Tickets
GET /api/tickets
POST /api/tickets
PUT /api/tickets/:id
GET /api/tickets/classify/:id  // AI classification

// Analytics
GET /api/analytics/dashboard
GET /api/analytics/user/:id
GET /api/analytics/risks
GET /api/analytics/burnout
```

### Authentication Middleware Example

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization').replace('Bearer ', '');
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    req.userId = decoded.userId;
    req.userRole = decoded.role;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.userRole !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { auth, adminOnly };
```

### User Model Example

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  tasksCompleted: { type: Number, default: 0 },
  averageTaskTime: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(password) {
  return await bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Management Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { auth } = require('../middleware/auth');
const Task = require('../models/Task');

// Get all tasks for user
router.get('/', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.userId })
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create new task
router.post('/', auth, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      assignedTo: req.body.assignedTo || req.userId,
      createdBy: req.userId
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status (for Kanban board)
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.userId },
      { status, updatedAt: new Date() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Configuration

### Environment Variables

Create `ml-service/.env`:
```bash
ML_SERVICE_PORT=8000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

### Requirements

```txt
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
scikit-learn==1.3.2
river==0.19.0
pandas==2.1.3
numpy==1.24.3
pymongo==4.6.0
python-dotenv==1.0.0
pydantic==2.5.0
```

### AI Ticket Classification

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
        self.categories = ['Technical', 'HR', 'Finance', 'General']
        self.load_model()
    
    def load_model(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            with open(f'{model_path}/ticket_classifier.pkl', 'rb') as f:
                self.model = pickle.load(f)
            with open(f'{model_path}/vectorizer.pkl', 'rb') as f:
                self.vectorizer = pickle.load(f)
        except FileNotFoundError:
            print("Model not found, will train on first use")
    
    def predict(self, ticket_text):
        features = self.vectorizer.transform([ticket_text])
        prediction = self.model.predict(features)[0]
        probabilities = self.model.predict_proba(features)[0]
        
        return {
            'category': self.categories[prediction],
            'confidence': float(max(probabilities)),
            'all_scores': {
                cat: float(prob) 
                for cat, prob in zip(self.categories, probabilities)
            }
        }
    
    def train(self, tickets, labels):
        X = self.vectorizer.fit_transform(tickets)
        self.model.fit(X, labels)
        self.save_model()
    
    def save_model(self):
        model_path = os.getenv('MODEL_PATH', './models')
        os.makedirs(model_path, exist_ok=True)
        
        with open(f'{model_path}/ticket_classifier.pkl', 'wb') as f:
            pickle.dump(self.model, f)
        with open(f'{model_path}/vectorizer.pkl', 'wb') as f:
            pickle.dump(self.vectorizer, f)
```

### Anomaly Detection Service

```python
# ml-service/services/anomaly_detector.py
from river import anomaly
from datetime import datetime
import numpy as np

class AnomalyDetector:
    def __init__(self):
        # Use Half-Space Trees for streaming anomaly detection
        self.model = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)
        
    def detect_user_anomaly(self, user_activity):
        """
        Detect anomalies in user behavior
        
        Args:
            user_activity: dict with keys like login_time, tasks_created, 
                          failed_logins, data_accessed, etc.
        """
        features = {
            'hour_of_day': user_activity.get('hour_of_day', 0),
            'tasks_created': user_activity.get('tasks_created', 0),
            'failed_logins': user_activity.get('failed_logins', 0),
            'data_volume': user_activity.get('data_volume', 0),
            'session_duration': user_activity.get('session_duration', 0),
        }
        
        # Get anomaly score (higher = more anomalous)
        score = self.model.score_one(features)
        
        # Update model with new observation
        self.model.learn_one(features)
        
        return {
            'is_anomaly': score > 0.7,  # Threshold
            'anomaly_score': float(score),
            'risk_level': self._calculate_risk_level(score)
        }
    
    def _calculate_risk_level(self, score):
        if score > 0.9:
            return 'critical'
        elif score > 0.7:
            return 'high'
        elif score > 0.5:
            return 'medium'
        return 'low'
```

### Burnout Detection

```python
# ml-service/services/burnout_detector.py
from datetime import datetime, timedelta
import numpy as np

class BurnoutDetector:
    def __init__(self):
        self.burnout_threshold = 0.7
    
    def analyze_user_burnout(self, user_data):
        """
        Analyze user workload and predict burnout risk
        
        Args:
            user_data: dict with task metrics, work hours, completion rates
        """
        # Extract metrics
        avg_work_hours = user_data.get('avg_work_hours_per_day', 8)
        tasks_in_progress = user_data.get('tasks_in_progress', 0)
        overdue_tasks = user_data.get('overdue_tasks', 0)
        completion_rate = user_data.get('completion_rate', 1.0)
        consecutive_work_days = user_data.get('consecutive_work_days', 0)
        
        # Calculate burnout score (0-1)
        workload_score = min(tasks_in_progress / 20, 1.0)  # Normalize to 20 tasks
        overtime_score = max(0, (avg_work_hours - 8) / 4)  # Hours over 8
        overdue_score = min(overdue_tasks / 10, 1.0)
        completion_score = 1 - completion_rate  # Lower completion = higher risk
        fatigue_score = min(consecutive_work_days / 14, 1.0)  # 2 weeks = max
        
        burnout_score = np.mean([
            workload_score * 0.3,
            overtime_score * 0.25,
            overdue_score * 0.2,
            completion_score * 0.15,
            fatigue_score * 0.1
        ])
        
        return {
            'burnout_score': float(burnout_score),
            'risk_level': self._get_risk_level(burnout_score),
            'recommendations': self._get_recommendations(burnout_score, user_data),
            'metrics': {
                'workload': float(workload_score),
                'overtime': float(overtime_score),
                'overdue': float(overdue_score),
                'completion': float(completion_score),
                'fatigue': float(fatigue_score)
            }
        }
    
    def _get_risk_level(self, score):
        if score > 0.8:
            return 'critical'
        elif score > 0.6:
            return 'high'
        elif score > 0.4:
            return 'medium'
        return 'low'
    
    def _get_recommendations(self, score, user_data):
        recommendations = []
        
        if score > 0.7:
            recommendations.append('Immediate workload reduction needed')
        if user_data.get('avg_work_hours_per_day', 8) > 10:
            recommendations.append('Reduce overtime hours')
        if user_data.get('consecutive_work_days', 0) > 10:
            recommendations.append('Schedule time off')
        if user_data.get('overdue_tasks', 0) > 5:
            recommendations.append('Reassign or postpone some tasks')
        
        return recommendations
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, List, Optional
import os
from dotenv import load_dotenv

from services.ticket_classifier import TicketClassifier
from services.anomaly_detector import AnomalyDetector
from services.burnout_detector import BurnoutDetector

load_dotenv()

app = FastAPI(title="Enterprise User Management AI Service")

# Initialize AI services
ticket_classifier = TicketClassifier()
anomaly_detector = AnomalyDetector()
burnout_detector = BurnoutDetector()

# Request models
class TicketClassifyRequest(BaseModel):
    text: str

class UserActivityRequest(BaseModel):
    hour_of_day: int
    tasks_created: int
    failed_logins: int = 0
    data_volume: int = 0
    session_duration: int = 0

class BurnoutAnalysisRequest(BaseModel):
    avg_work_hours_per_day: float
    tasks_in_progress: int
    overdue_tasks: int
    completion_rate: float
    consecutive_work_days: int

@app.get("/")
def read_root():
    return {"message": "Enterprise User Management AI Service", "status": "running"}

@app.post("/classify-ticket")
def classify_ticket(request: TicketClassifyRequest):
    """Classify support ticket using AI"""
    try:
        result = ticket_classifier.predict(request.text)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
def detect_anomaly(request: UserActivityRequest):
    """Detect anomalies in user behavior"""
    try:
        result = anomaly_detector.detect_user_anomaly(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze user burnout risk"""
    try:
        result = burnout_detector.analyze_user_burnout(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend Integration

### API Service Setup

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_BASE_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add JWT token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authAPI = {
  login: (credentials) => api.post('/api/auth/login', credentials),
  register: (userData) => api.post('/api/auth/register', userData),
  getCurrentUser: () => api.get('/api/auth/me')
};

export const taskAPI = {
  getTasks: () => api.get('/api/tasks'),
  createTask: (task) => api.post('/api/tasks', task),
  updateTaskStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  deleteTask: (id) => api.delete(`/api/tasks/${id}`)
};

export const ticketAPI = {
  getTickets: () => api.get('/api/tickets'),
  createTicket: (ticket) => api.post('/api/tickets', ticket),
  classifyTicket: async (ticketId, text) => {
    const mlResponse = await axios.post(`${ML_BASE_URL}/classify-ticket`, { text });
    return api.put(`/api/tickets/${ticketId}`, { 
      category: mlResponse.data.category,
      aiConfidence: mlResponse.data.confidence
    });
  }
};

export const analyticsAPI = {
  getDashboard: () => api.get('/api/analytics/dashboard'),
  detectAnomaly: (activityData) => axios.post(`${ML_BASE_URL}/detect-anomaly`, activityData),
  analyzeBurnout: (userData) => axios.post(`${ML_BASE_URL}/analyze-burnout`, userData)
};

export default api;
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getTasks();
      setTasks(response.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    try {
      await taskAPI.updateTaskStatus(taskId, newStatus);
      
      // Update local state
      setTasks(tasks.map(task => 
        task._id === taskId ? { ...task, status: newStatus } : task
      ));
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const columns = ['To Do', 'In Progress', 'Done'];

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {columns.map(column => (
        <div 
          key={column}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, column)}
          onDragOver={handleDragOver}
        >
          <h3>{column}</h3>
          <div className="task-list">
            {tasks
              .filter(task => task.status === column)
              .map(task => (
                <div
                  key={task._id}
                  className="task-card"
                  draggable
                  onDragStart={(e) => handleDragStart(e, task._id)}
                >
                  <h4>{task.title}</h4>
                  <p>{task.description}</p>
                  <span className="task-priority">{task.priority}</span>
                </div>
              ))
            }
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { analyticsAPI } from '../services/api';
import './AdminDashboard.css';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [burnoutRisks, setBurnoutRisks] = useState([]);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const response = await analyticsAPI.getDashboard();
      setAnalytics(response.data);
      
      // Check burnout for each user
      const burnoutChecks = await Promise.all(
        response.data.users.map(async (user) => {
          const burnoutData = {
            avg_work_hours_per_day: user.avgWorkHours || 8,
            tasks_in_progress: user.activeTasks || 0,
            overdue_tasks: user.overdueTasks || 0,
            completion_rate: user.completionRate || 1.0,
            consecutive_work_days: user.consecutiveDays || 0
          };
          
          const result = await analyticsAPI.analyzeBurnout(burnoutData);
          return { userId: user._id, userName: user.name, ...result.data };
        })
      );
      
      setBurnoutRisks(burnoutChecks.filter(r => r.risk_level !== 'low'));
    } catch (error) {
      console.error('Error fetching dashboard:', error);
    }
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{analytics.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p className="stat-value">{analytics.activeTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-value">{analytics.openTickets}</p>
        </div>
        <div className="stat-card">
          <h3>Completion Rate</h3>
          <p className="stat-value">{(analytics.completionRate * 100).toFixed(1)}%</p>
        </div>
      </div>

      <div className="burnout-alerts">
        <h2>Burnout Risk Alerts</h2>
        {burnoutRisks.length === 0 ? (
          <p>No high burnout risks detected</p>
        ) : (
          <div className="risk-list">
            {burnoutRisks.map(risk => (
              <div key={risk.userId} className={`risk-card ${risk.risk_level}`}>
                <h4>{risk.userName}</h4>
                <p>Risk Level: <strong>{risk.risk_level.toUpperCase()}</strong></p>
                <p>Burnout Score: {(risk.burnout_score * 100).toFixed(1)}%</p>
                <ul>
                  {risk.recommendations.map((rec, idx) => (
                    <li key={idx}>{rec}</li>
                  ))}
                </ul>
              </div>
            ))}
          </div>
        )}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Common Patterns

### JWT Authentication Flow

```javascript
// Login and store token
const handleLogin = async (email, password) => {
  try {
    const response = await authAPI.login({ email, password });
    const { token, user } = response.data;
    
    // Store token
    localStorage.setItem('token', token);
    localStorage.setItem('user', JSON.stringify(user));
    
    // Redirect based on role
    if (user.role === 'admin') {
      navigate('/admin/dashboard');
    } else {
      navigate('/user/dashboard');
    }
  } catch (error) {
    console.error('Login failed:', error);
  }
};
```

### Real-time Anomaly Detection

```javascript
// Track user activity and detect anomalies
const trackUserActivity = async () => {
  const activityData = {
    hour_of_day: new Date().getHours(),
    tasks_created: tasksCreatedToday,
    failed_logins: failedLoginAttempts,
    data_volume: dataAccessedMB,
    session_duration: sessionDurationMinutes
  };
  
  try {
    const result = await analyticsAPI.detectAnomaly(activityData);
    
    if (result.data.is_anomaly && result.data.risk_level === 'critical') {
      // Alert admin
      await sendAdminAlert({
        userId: currentUser.id,
        anomalyScore: result.data.anomaly_score,
        riskLevel: result.data.risk_level
      });
    }
  } catch (error) {
    console.error('Anomaly detection failed:', error);
  }
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check if MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Verify connection
mongo --eval "db.adminCommand('ping')"
```

### JWT Token Expiration

```javascript
// Add token refresh logic
api.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      // Token expired, redirect to login
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### ML Service Model Not Found

```bash
# Create models directory
mkdir -p ml-service/models

# Train initial model (run from ml-service directory)
python scripts/train_initial_models.py
```

### CORS Issues

```javascript
// backend/index.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Port Already in Use

```bash
# Find and kill process on port 5000
lsof -ti:5000 | xargs kill -9

# Or use different port
PORT=5001 npm start
```

## Testing

### Test ML Service Endpoints

```bash
# Test ticket classification
curl -X POST http://localhost:8000/classify-ticket \
  -H "Content-Type: application/json" \
  -d '{"text": "My computer is not starting up properly"}'

# Test anomaly detection
curl -X POST http://localhost:8000/detect-anomaly \
  -H "Content-Type: application/json" \
  -d '{
    "hour_of_day": 3,
    "tasks_created": 50,
    "failed_logins": 10,
    "data_volume": 5000,
    "session_duration": 300
  }'

# Test burnout analysis
curl -X POST http://localhost:8000/analyze-burnout \
  -H "Content-Type: application/json" \
  -d '{
    "avg_work_hours_per_day": 12,
    "tasks_in_progress": 25,
    "overdue_tasks": 8,
    "completion_rate": 0.6,
    "consecutive_work_days": 15
  }'
```

### Test Backend API

```bash
# Register user
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test User",
    "email": "test@example.com",
    "password": "password123",
    "role": "user"
  }'

# Login and get token
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "password123"
  }'

# Create task (use token from login)
curl -X POST http://localhost:5000/api/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{
    "title": "Complete documentation",
    "description": "Write comprehensive docs",
    "priority": "high",
    "status": "To Do"
  }'
```

This skill provides comprehensive guidance for working with the Enterprise
