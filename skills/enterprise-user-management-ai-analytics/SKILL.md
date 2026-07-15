---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI-powered ticket classification"
  - "build user management with risk detection"
  - "integrate ML service for burnout analysis"
  - "implement Kanban board for task management"
  - "set up JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system that combines user administration, task tracking, and support ticket management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Management**: Kanban-style task boards with time tracking and progress monitoring
- **Support System**: Ticket management with AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, and user monitoring

## Installation

### Prerequisites

- Node.js (v14+)
- Python 3.8+
- MongoDB

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
MONGODB_URI=your_mongodb_connection_string
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

Create `.env` file in ml-service directory:

```env
MODEL_PATH=./models
LOG_LEVEL=info
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
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
Body: { "email": "user@example.com", "password": "password123" }
Response: { "token": "jwt_token", "user": {...} }

// Register
POST /api/auth/register
Body: { "name": "John Doe", "email": "john@example.com", "password": "password123", "role": "user" }
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Create user
POST /api/users
Headers: { "Authorization": "Bearer <token>" }
Body: { "name": "Jane Doe", "email": "jane@example.com", "role": "user" }

// Update user
PUT /api/users/:id
Headers: { "Authorization": "Bearer <token>" }
Body: { "name": "Jane Smith", "status": "active" }

// Delete user
DELETE /api/users/:id
Headers: { "Authorization": "Bearer <token>" }
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { "Authorization": "Bearer <token>" }

// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer <token>" }
Body: { 
  "title": "Complete feature",
  "description": "Build user dashboard",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo"
}

// Update task status
PUT /api/tasks/:id
Headers: { "Authorization": "Bearer <token>" }
Body: { "status": "in_progress", "timeSpent": 120 }
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer <token>" }
Body: { 
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high"
}

// Get tickets
GET /api/tickets
Headers: { "Authorization": "Bearer <token>" }

// Update ticket
PUT /api/tickets/:id
Headers: { "Authorization": "Bearer <token>" }
Body: { "status": "resolved" }
```

### AI Analytics Endpoints

```javascript
// Risk prediction
POST /api/ml/risk-prediction
Headers: { "Authorization": "Bearer <token>" }
Body: { "userId": "user_id" }
Response: { "riskScore": 0.75, "factors": ["high_workload", "missed_deadlines"] }

// Anomaly detection
POST /api/ml/anomaly-detection
Body: { "userId": "user_id", "activityData": [...] }
Response: { "isAnomaly": true, "anomalyScore": 0.85 }

// Burnout analysis
POST /api/ml/burnout-analysis
Body: { "userId": "user_id", "workloadData": {...} }
Response: { "burnoutRisk": "high", "score": 0.82, "recommendations": [...] }

// Project delay prediction
POST /api/ml/project-prediction
Body: { "projectId": "project_id", "taskData": [...] }
Response: { "delayProbability": 0.65, "estimatedDelay": 5 }
```

## Code Examples

### Frontend: User Authentication

```javascript
// src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const login = async (email, password) => {
  try {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  } catch (error) {
    throw error.response.data;
  }
};

export const logout = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
};

export const getCurrentUser = () => {
  return JSON.parse(localStorage.getItem('user'));
};

export const getAuthHeader = () => {
  const token = localStorage.getItem('token');
  return token ? { Authorization: `Bearer ${token}` } : {};
};
```

### Frontend: Task Management Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks`, {
        headers: getAuthHeader()
      });
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(
        `${API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: getAuthHeader() }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, status) => {
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div 
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={(e) => e.preventDefault()}
        >
          <h3>{status === 'inProgress' ? 'In Progress' : status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              className="task-card"
              draggable
              onDragStart={(e) => handleDragStart(e, task._id)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority ${task.priority}`}>{task.priority}</span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Backend: User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Get all users (Admin only)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

// Create user
exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = new User({ name, email, password, role });
    await user.save();

    res.status(201).json({
      message: 'User created successfully',
      user: { id: user._id, name: user.name, email: user.email, role: user.role }
    });
  } catch (error) {
    res.status(500).json({ message: 'Error creating user', error: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    const user = await User.findByIdAndUpdate(
      id,
      updates,
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ message: 'User updated successfully', user });
  } catch (error) {
    res.status(500).json({ message: 'Error updating user', error: error.message });
  }
};

// Delete user
exports.deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    
    const user = await User.findByIdAndDelete(id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Error deleting user', error: error.message });
  }
};
```

### Backend: Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId).select('-password');

    if (!user) {
      return res.status(401).json({ message: 'User not found' });
    }

    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid authentication token' });
  }
};

exports.isAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};
```

### ML Service: Risk Prediction

```python
# ml-service/services/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import pickle
import os

class RiskPredictor:
    def __init__(self):
        self.model = None
        self.load_model()
    
    def load_model(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            with open(f'{model_path}/risk_model.pkl', 'rb') as f:
                self.model = pickle.load(f)
        except FileNotFoundError:
            # Initialize new model if none exists
            self.model = RandomForestClassifier(n_estimators=100)
    
    def extract_features(self, user_data):
        """Extract features from user activity data"""
        features = [
            user_data.get('task_completion_rate', 0),
            user_data.get('avg_delay_days', 0),
            user_data.get('total_tasks', 0),
            user_data.get('overdue_tasks', 0),
            user_data.get('hours_worked', 0),
            user_data.get('ticket_count', 0)
        ]
        return np.array(features).reshape(1, -1)
    
    def predict_risk(self, user_data):
        """Predict risk score for a user"""
        features = self.extract_features(user_data)
        
        if hasattr(self.model, 'predict_proba'):
            risk_score = self.model.predict_proba(features)[0][1]
        else:
            risk_score = 0.5  # Default score if model not trained
        
        # Identify risk factors
        factors = []
        if user_data.get('task_completion_rate', 100) < 70:
            factors.append('low_completion_rate')
        if user_data.get('overdue_tasks', 0) > 3:
            factors.append('multiple_overdue_tasks')
        if user_data.get('hours_worked', 0) > 50:
            factors.append('high_workload')
        
        return {
            'riskScore': float(risk_score),
            'riskLevel': 'high' if risk_score > 0.7 else 'medium' if risk_score > 0.4 else 'low',
            'factors': factors
        }
```

### ML Service: FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
from services.risk_predictor import RiskPredictor
from services.anomaly_detector import AnomalyDetector
from services.burnout_analyzer import BurnoutAnalyzer
import os

app = FastAPI(title="User Management ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize services
risk_predictor = RiskPredictor()
anomaly_detector = AnomalyDetector()
burnout_analyzer = BurnoutAnalyzer()

# Request models
class RiskPredictionRequest(BaseModel):
    userId: str
    task_completion_rate: Optional[float] = 0
    avg_delay_days: Optional[float] = 0
    total_tasks: Optional[int] = 0
    overdue_tasks: Optional[int] = 0
    hours_worked: Optional[float] = 0
    ticket_count: Optional[int] = 0

class AnomalyDetectionRequest(BaseModel):
    userId: str
    activityData: List[Dict]

class BurnoutRequest(BaseModel):
    userId: str
    workloadData: Dict

@app.get("/")
def root():
    return {"message": "User Management ML Service", "version": "1.0"}

@app.post("/api/ml/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    try:
        user_data = request.dict()
        result = risk_predictor.predict_risk(user_data)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        result = anomaly_detector.detect(request.activityData)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutRequest):
    try:
        result = burnout_analyzer.analyze(request.workloadData)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

## Common Patterns

### Protected Routes Pattern

```javascript
// frontend/src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { getCurrentUser } from '../services/authService';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const user = getCurrentUser();
  
  if (!user) {
    return <Navigate to="/login" />;
  }
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

<Routes>
  <Route path="/login" element={<Login />} />
  <Route path="/dashboard" element={
    <ProtectedRoute>
      <Dashboard />
    </ProtectedRoute>
  } />
  <Route path="/admin" element={
    <ProtectedRoute adminOnly={true}>
      <AdminPanel />
    </ProtectedRoute>
  } />
</Routes>
```

### Real-time Notifications Pattern

```javascript
// frontend/src/hooks/useNotifications.js
import { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

export const useNotifications = () => {
  const [notifications, setNotifications] = useState([]);
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000); // Poll every 30s
    return () => clearInterval(interval);
  }, []);

  const fetchNotifications = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/notifications`, {
        headers: getAuthHeader()
      });
      setNotifications(response.data);
    } catch (error) {
      console.error('Error fetching notifications:', error);
    }
  };

  const markAsRead = async (notificationId) => {
    try {
      await axios.put(
        `${API_URL}/api/notifications/${notificationId}`,
        { read: true },
        { headers: getAuthHeader() }
      );
      fetchNotifications();
    } catch (error) {
      console.error('Error marking notification as read:', error);
    }
  };

  return { notifications, markAsRead, refresh: fetchNotifications };
};
```

### AI Analytics Dashboard Pattern

```javascript
// frontend/src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);
  const ML_URL = process.env.REACT_APP_ML_URL;

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    setLoading(true);
    try {
      const user = JSON.parse(localStorage.getItem('user'));
      
      // Fetch user data for ML analysis
      const userDataResponse = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/users/${user.id}/analytics`,
        { headers: getAuthHeader() }
      );

      // Get AI predictions
      const [riskResponse, burnoutResponse] = await Promise.all([
        axios.post(`${ML_URL}/api/ml/risk-prediction`, {
          userId: user.id,
          ...userDataResponse.data
        }),
        axios.post(`${ML_URL}/api/ml/burnout-analysis`, {
          userId: user.id,
          workloadData: userDataResponse.data
        })
      ]);

      setAnalytics({
        risk: riskResponse.data,
        burnout: burnoutResponse.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="risk-analysis">
        <h3>Risk Analysis</h3>
        <p>Risk Level: {analytics?.risk?.riskLevel}</p>
        <p>Score: {(analytics?.risk?.riskScore * 100).toFixed(2)}%</p>
        <ul>
          {analytics?.risk?.factors.map((factor, i) => (
            <li key={i}>{factor}</li>
          ))}
        </ul>
      </div>
      
      <div className="burnout-analysis">
        <h3>Burnout Analysis</h3>
        <p>Risk: {analytics?.burnout?.burnoutRisk}</p>
        <p>Score: {(analytics?.burnout?.score * 100).toFixed(2)}%</p>
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Configuration

### MongoDB Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: { type: Date }
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

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { 
    type: String, 
    enum: ['todo', 'in_progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  dueDate: { type: Date },
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

### Environment Variables Reference

```env
# Backend (.env)
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user_management
JWT_SECRET=your_secret_key_here
JWT_EXPIRATION=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development

# Frontend (.env)
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000

# ML Service (.env)
MODEL_PATH=./models
LOG_LEVEL=info
PYTHONUNBUFFERED=1
```

## Troubleshooting

### JWT Authentication Issues

```javascript
// Check token expiration
const jwt = require('jsonwebtoken');

const verifyToken = (token) => {
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    console.log('Token valid until:', new Date(decoded.exp * 1000));
    return decoded;
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      console.error('Token expired at:', error.expiredAt);
    } else if (error.name === 'JsonWebTokenError') {
      console.error('Invalid token:', error.message);
    }
    return null;
  }
};
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const options = {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    };
    
    await mongoose.connect(process.env.MONGODB_URI, options);
    console.log('MongoDB connected successfully');
    
    mongoose.connection.on('error', (err) => {
      console.error('MongoDB connection error:', err);
    });
    
    mongoose.connection.on('disconnected', () => {
      console.warn('MongoDB disconnected. Attempting to reconnect...');
    });
  } catch (error) {
    console.error('MongoDB connection failed:', error.message);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

const corsOptions = {
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));
```

### ML Service Model Loading

```python
# ml-service/services/model_manager.py
import pickle
import os
import logging

logger = logging.getLogger(__name__)

def load_model_safely(model_path, model_name):
    """Safely load ML model with error handling"""
    full_path = os.path.join(model_path, f'{model_name}.pkl')
    
    try:
        if os.path.exists(full_path):
            with open(full_path, 'rb') as f:
                model = pickle.load(f)
            logger.info(f"Model {model_name} loaded successfully")
            return model
        else:
            logger.warning(f"Model file not found: {full_path}")
            return None
    except Exception as e:
        logger.error(f"Error loading model {model_name}: {str(e)}")
        return None

def save_model(model, model_path, model_name):
    """Save trained model"""
    os.makedirs(model_path, exist_ok=True)
    full_path = os.path.join(model_path, f'{model_name}.pkl')
    
    try:
        with open(full_path, 'wb') as f:
            pickle.dump(model, f)
        logger.info(f"Model {model_name} saved successfully")
    except Exception as e:
        logger.error(f"Error saving model {model_name}: {str(e)}")
```

### Frontend API Error Handling

```javascript
// frontend/src/services/apiClient.js
import axios from 'axios';
import { getAuthHeader, logout } from './authService';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Request interceptor
apiClient.interceptors.request.use(
  (config) => {
    const headers = getAuthHeader();
    config.headers = { ...config.headers, ...headers };
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Token expired or invalid
      logout();
      window.location.href = '/login';
    } else if (error.response?.status === 403) {
      console.error('Access denied');
    } else if (error.response?.status >= 500) {
      console.error('Server error:', error.response.data);
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

This skill provides comprehensive guidance for working with the Enterprise User Management System with AI Analytics, covering authentication, user management, task tracking, AI-powered analytics, and troubleshooting common issues.
