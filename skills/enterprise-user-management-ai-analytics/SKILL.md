---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI-based ticket classification"
  - "build user management with AI analytics"
  - "configure task tracking with anomaly detection"
  - "create admin dashboard with AI insights"
  - "add burnout detection to user management"
  - "integrate ML service for risk prediction"
  - "deploy user management system with AI"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system combining React frontend, Node.js backend, and FastAPI ML service for intelligent task management, ticket routing, risk detection, and workforce analytics.

## What This Project Does

This system provides:
- **User Management**: Role-based access control, authentication via JWT
- **Task Tracking**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide insights, audit logs, performance metrics

## Installation

### Prerequisites
- Node.js 14+
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

Create `backend/.env`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
```

Start backend:
```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `ml-service/.env`:
```env
MODEL_PATH=./models
BACKEND_URL=http://localhost:5000
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
# Runs at http://localhost:3000
```

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securepass123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Backend)

```javascript
// GET /api/users - Get all users (admin only)
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (admin only)
```

### Task Management (Backend)

```javascript
// POST /api/tasks
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// GET /api/tasks - Get all tasks
// GET /api/tasks/user/:userId - Get tasks for user
// PUT /api/tasks/:id - Update task
// PATCH /api/tasks/:id/status - Update task status
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets
{
  "title": "Login issue",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical"
}

// GET /api/tickets - Get all tickets
// PUT /api/tickets/:id - Update ticket
```

### AI Analytics (ML Service)

```python
# POST /api/ml/classify-ticket
{
  "title": "Password reset not working",
  "description": "I tried resetting my password but didn't receive email"
}
# Returns: { "category": "technical", "priority": "medium", "confidence": 0.87 }

# POST /api/ml/predict-risk
{
  "userId": "user_id",
  "failedLogins": 5,
  "afterHoursActivity": 12,
  "dataAccessCount": 150
}
# Returns: { "riskScore": 0.78, "riskLevel": "high" }

# POST /api/ml/detect-burnout
{
  "userId": "user_id",
  "tasksCompleted": 45,
  "hoursWorked": 65,
  "overtimeHours": 15,
  "tasksOverdue": 8
}
# Returns: { "burnoutScore": 0.72, "level": "moderate", "recommendations": [...] }

# POST /api/ml/predict-delay
{
  "projectId": "proj_123",
  "completedTasks": 20,
  "totalTasks": 50,
  "daysElapsed": 30,
  "totalDays": 60
}
# Returns: { "delayProbability": 0.65, "estimatedDelay": 10 }
```

## Frontend Integration

### Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    setToken(response.data.token);
    setUser(response.data.user);
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

export default AuthContext;
```

### Task Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    const fetchTasks = async () => {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`
      );
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
    };
    
    fetchTasks();
  }, [userId]);

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      { status: newStatus }
    );
    // Refresh tasks
  };

  return (
    <div className="task-board">
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
        ))}
      </div>
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
        ))}
      </div>
      <div className="column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
        ))}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### AI Ticket Classification

```javascript
// frontend/src/services/aiService.js
import axios from 'axios';

const ML_API = process.env.REACT_APP_ML_URL;

export const classifyTicket = async (title, description) => {
  try {
    const response = await axios.post(`${ML_API}/api/ml/classify-ticket`, {
      title,
      description
    });
    return response.data;
  } catch (error) {
    console.error('Ticket classification failed:', error);
    throw error;
  }
};

export const detectRisk = async (userData) => {
  const response = await axios.post(`${ML_API}/api/ml/predict-risk`, userData);
  return response.data;
};

export const analyzeBurnout = async (workloadData) => {
  const response = await axios.post(`${ML_API}/api/ml/detect-burnout`, workloadData);
  return response.data;
};

// Usage in component
import { classifyTicket } from '../services/aiService';

const TicketForm = () => {
  const [formData, setFormData] = useState({ title: '', description: '' });
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    
    // Get AI classification
    const classification = await classifyTicket(
      formData.title,
      formData.description
    );
    
    // Submit ticket with AI-suggested category and priority
    await axios.post(`${process.env.REACT_APP_API_URL}/api/tickets`, {
      ...formData,
      category: classification.category,
      priority: classification.priority
    });
  };
};
```

## Backend Implementation

### User Model (MongoDB)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  failedLoginAttempts: { type: Number, default: 0 }
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

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error('No token provided');
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);
    
    if (!user) {
      throw new Error('User not found');
    }
    
    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
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

## ML Service Implementation

### Ticket Classification Model

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import pickle
import os

class TicketClassifier:
    def __init__(self, model_path='./models'):
        self.model_path = model_path
        self.vectorizer = TfidfVectorizer(max_features=1000)
        self.category_model = MultinomialNB()
        self.priority_model = MultinomialNB()
        
        if os.path.exists(f'{model_path}/ticket_classifier.pkl'):
            self.load_model()
    
    def train(self, texts, categories, priorities):
        X = self.vectorizer.fit_transform(texts)
        self.category_model.fit(X, categories)
        self.priority_model.fit(X, priorities)
        self.save_model()
    
    def predict(self, text):
        X = self.vectorizer.transform([text])
        category = self.category_model.predict(X)[0]
        priority = self.priority_model.predict(X)[0]
        
        cat_proba = max(self.category_model.predict_proba(X)[0])
        
        return {
            'category': category,
            'priority': priority,
            'confidence': float(cat_proba)
        }
    
    def save_model(self):
        os.makedirs(self.model_path, exist_ok=True)
        with open(f'{self.model_path}/ticket_classifier.pkl', 'wb') as f:
            pickle.dump({
                'vectorizer': self.vectorizer,
                'category_model': self.category_model,
                'priority_model': self.priority_model
            }, f)
    
    def load_model(self):
        with open(f'{self.model_path}/ticket_classifier.pkl', 'rb') as f:
            data = pickle.load(f)
            self.vectorizer = data['vectorizer']
            self.category_model = data['category_model']
            self.priority_model = data['priority_model']
```

### Risk Detection

```python
# ml-service/models/risk_detector.py
import numpy as np
from sklearn.ensemble import RandomForestClassifier

class RiskDetector:
    def __init__(self):
        self.model = RandomForestClassifier(n_estimators=100)
        self.threshold_high = 0.7
        self.threshold_medium = 0.4
    
    def calculate_risk_score(self, features):
        # Features: failed_logins, after_hours_activity, data_access_count, 
        # permission_changes, login_location_changes
        
        weights = np.array([0.3, 0.2, 0.2, 0.15, 0.15])
        
        # Normalize features
        normalized = np.array([
            min(features['failedLogins'] / 10, 1),
            min(features['afterHoursActivity'] / 20, 1),
            min(features['dataAccessCount'] / 200, 1),
            min(features.get('permissionChanges', 0) / 5, 1),
            min(features.get('locationChanges', 0) / 3, 1)
        ])
        
        risk_score = np.dot(normalized, weights)
        
        if risk_score >= self.threshold_high:
            risk_level = 'high'
        elif risk_score >= self.threshold_medium:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        return {
            'riskScore': float(risk_score),
            'riskLevel': risk_level,
            'factors': {
                'failedLogins': features['failedLogins'],
                'afterHoursActivity': features['afterHoursActivity'],
                'dataAccess': features['dataAccessCount']
            }
        }
```

### Burnout Analysis

```python
# ml-service/models/burnout_analyzer.py
class BurnoutAnalyzer:
    def analyze(self, workload_data):
        # Extract features
        tasks_completed = workload_data['tasksCompleted']
        hours_worked = workload_data['hoursWorked']
        overtime_hours = workload_data['overtimeHours']
        tasks_overdue = workload_data['tasksOverdue']
        
        # Calculate burnout indicators
        workload_score = min(hours_worked / 40, 2)  # 40 is baseline
        overtime_score = min(overtime_hours / 10, 2)
        overdue_score = min(tasks_overdue / 5, 2)
        
        # Combined burnout score
        burnout_score = (workload_score * 0.4 + 
                        overtime_score * 0.4 + 
                        overdue_score * 0.2)
        
        # Normalize to 0-1
        burnout_score = min(burnout_score / 2, 1)
        
        if burnout_score >= 0.7:
            level = 'high'
            recommendations = [
                'Reduce workload immediately',
                'Schedule time off',
                'Redistribute tasks'
            ]
        elif burnout_score >= 0.4:
            level = 'moderate'
            recommendations = [
                'Monitor workload closely',
                'Limit overtime hours',
                'Regular check-ins'
            ]
        else:
            level = 'low'
            recommendations = ['Maintain current pace']
        
        return {
            'burnoutScore': float(burnout_score),
            'level': level,
            'recommendations': recommendations,
            'metrics': {
                'hoursWorked': hours_worked,
                'overtimeHours': overtime_hours,
                'tasksOverdue': tasks_overdue
            }
        }
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.ticket_classifier import TicketClassifier
from models.risk_detector import RiskDetector
from models.burnout_analyzer import BurnoutAnalyzer

app = FastAPI()

ticket_classifier = TicketClassifier()
risk_detector = RiskDetector()
burnout_analyzer = BurnoutAnalyzer()

class TicketRequest(BaseModel):
    title: str
    description: str

class RiskRequest(BaseModel):
    userId: str
    failedLogins: int
    afterHoursActivity: int
    dataAccessCount: int

class BurnoutRequest(BaseModel):
    userId: str
    tasksCompleted: int
    hoursWorked: float
    overtimeHours: float
    tasksOverdue: int

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    try:
        text = f"{request.title} {request.description}"
        result = ticket_classifier.predict(text)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskRequest):
    try:
        features = {
            'failedLogins': request.failedLogins,
            'afterHoursActivity': request.afterHoursActivity,
            'dataAccessCount': request.dataAccessCount
        }
        result = risk_detector.calculate_risk_score(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    try:
        workload_data = {
            'tasksCompleted': request.tasksCompleted,
            'hoursWorked': request.hoursWorked,
            'overtimeHours': request.overtimeHours,
            'tasksOverdue': request.tasksOverdue
        }
        result = burnout_analyzer.analyze(workload_data)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

## Common Patterns

### Admin Dashboard Data Aggregation

```javascript
// backend/controllers/analyticsController.js
exports.getAdminAnalytics = async (req, res) => {
  try {
    const [userCount, taskStats, ticketStats, riskAlerts] = await Promise.all([
      User.countDocuments({ status: 'active' }),
      Task.aggregate([
        {
          $group: {
            _id: '$status',
            count: { $sum: 1 }
          }
        }
      ]),
      Ticket.aggregate([
        {
          $group: {
            _id: '$priority',
            count: { $sum: 1 }
          }
        }
      ]),
      // Fetch high-risk users from ML service
      axios.get(`${process.env.ML_SERVICE_URL}/api/ml/risk-alerts`)
    ]);

    res.json({
      users: { active: userCount },
      tasks: taskStats,
      tickets: ticketStats,
      riskAlerts: riskAlerts.data
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### Real-time Notifications with Socket.io

```javascript
// backend/sockets/notificationSocket.js
const socketIO = require('socket.io');

const initializeSocket = (server) => {
  const io = socketIO(server, {
    cors: { origin: process.env.FRONTEND_URL }
  });

  io.on('connection', (socket) => {
    console.log('User connected:', socket.id);

    socket.on('join-user-room', (userId) => {
      socket.join(`user-${userId}`);
    });

    socket.on('disconnect', () => {
      console.log('User disconnected:', socket.id);
    });
  });

  return io;
};

// Emit notification
const notifyUser = (io, userId, notification) => {
  io.to(`user-${userId}`).emit('notification', notification);
};

module.exports = { initializeSocket, notifyUser };
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Configuration

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Model Not Found

```python
# ml-service/main.py
@app.on_event("startup")
async def startup_event():
    # Initialize models with default training data if not found
    if not os.path.exists('./models/ticket_classifier.pkl'):
        print("Training default ticket classifier...")
        # Add default training logic here
```

### JWT Token Expiration

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Environment Variables Not Loading

```bash
# Ensure .env files are in correct locations and not in .gitignore
# backend/.env
# ml-service/.env
# frontend/.env

# Load them properly in Node.js
require('dotenv').config();

# In Python
from dotenv import load_dotenv
load_dotenv()
```

## Production Deployment

### Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'
services:
  mongodb:
    image: mongo:latest
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise_user_mgmt
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - mongodb

  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    volumes:
      - ./models:/app/models

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://backend:5000
      - REACT_APP_ML_URL=http://ml-service:8000

volumes:
  mongo-data:
```

Run with:
```bash
docker-compose up -d
```
