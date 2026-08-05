---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket classification, risk prediction, and burnout detection
triggers:
  - "build a user management system with AI analytics"
  - "create an enterprise task and ticket management platform"
  - "implement AI-powered user administration dashboard"
  - "set up user management with anomaly detection"
  - "build a task tracking system with burnout prediction"
  - "create admin dashboard with AI insights"
  - "implement kanban board with time tracking"
  - "set up ticket classification with machine learning"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for managing users, tasks, and support tickets with integrated AI analytics for risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

This system provides:
- **User Management**: Admin controls for user CRUD operations with role-based access
- **Task Management**: Kanban board with time tracking and progress monitoring
- **Ticket System**: Support ticket creation and AI-powered classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Authentication**: JWT-based secure login system
- **Dashboards**: Separate admin and user interfaces with real-time insights

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

Create `.env` file:
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

Create `.env` file:
```env
MODEL_PATH=./models
BACKEND_URL=http://localhost:5000
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

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
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
{
  "email": "user@example.com",
  "password": "password123"
}

// Register
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "password123",
  "role": "user"
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }

// Create user
POST /api/users
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
{
  "name": "Jane Smith Updated",
  "role": "admin"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/user/:userId
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }

// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Complete the user authentication module",
  "assignedTo": "user_id_here",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PUT /api/tasks/:taskId
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:taskId/time
{
  "timeSpent": 3600 // seconds
}
```

### Ticket Management

```javascript
// Create ticket
POST /api/tickets
{
  "subject": "Login issue",
  "description": "Cannot access my account",
  "priority": "high",
  "category": "technical"
}

// Get user tickets
GET /api/tickets/user/:userId

// Update ticket status
PUT /api/tickets/:ticketId
{
  "status": "resolved",
  "resolution": "Password reset link sent"
}
```

### AI Analytics Endpoints

```javascript
// Classify ticket (ML Service)
POST /ml/classify-ticket
{
  "subject": "Cannot login to system",
  "description": "Getting error when trying to access dashboard"
}
// Response: { "category": "technical", "priority": "high", "confidence": 0.89 }

// Predict risk
POST /ml/predict-risk
{
  "userId": "user_id_here",
  "failedLoginAttempts": 5,
  "dataAccessPatterns": [0.2, 0.8, 0.5],
  "workHours": [45, 50, 55]
}
// Response: { "riskScore": 0.72, "riskLevel": "high" }

// Detect burnout
POST /ml/detect-burnout
{
  "userId": "user_id_here",
  "tasksCompleted": 45,
  "averageTaskTime": 7200,
  "overtimeHours": 20,
  "weeklyWorkHours": [50, 55, 60, 58]
}
// Response: { "burnoutScore": 0.68, "recommendation": "Reduce workload" }

// Predict project delay
POST /ml/predict-delay
{
  "projectId": "project_id_here",
  "tasksTotal": 50,
  "tasksCompleted": 20,
  "daysElapsed": 30,
  "teamSize": 5
}
// Response: { "delayProbability": 0.75, "estimatedDays": 15 }
```

## Frontend Integration Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
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
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Board Component

```javascript
// src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`
      );
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="task-board">
      <Column 
        title="To Do" 
        tasks={tasks.todo} 
        onUpdateStatus={updateTaskStatus}
      />
      <Column 
        title="In Progress" 
        tasks={tasks.inProgress} 
        onUpdateStatus={updateTaskStatus}
      />
      <Column 
        title="Done" 
        tasks={tasks.done} 
        onUpdateStatus={updateTaskStatus}
      />
    </div>
  );
};

export default TaskBoard;
```

### AI-Powered Ticket Classification

```javascript
// src/components/CreateTicket.js
import React, { useState } from 'react';
import axios from 'axios';

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    subject: '',
    description: ''
  });
  const [aiSuggestion, setAiSuggestion] = useState(null);

  const classifyTicket = async () => {
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/ml/classify-ticket`,
        {
          subject: formData.subject,
          description: formData.description
        }
      );
      setAiSuggestion(response.data);
    } catch (error) {
      console.error('Error classifying ticket:', error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await axios.post(
        `${process.env.REACT_APP_API_URL}/api/tickets`,
        {
          ...formData,
          category: aiSuggestion?.category || 'general',
          priority: aiSuggestion?.priority || 'medium'
        }
      );
      alert('Ticket created successfully!');
    } catch (error) {
      console.error('Error creating ticket:', error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Subject"
        value={formData.subject}
        onChange={(e) => setFormData({ ...formData, subject: e.target.value })}
      />
      <textarea
        placeholder="Description"
        value={formData.description}
        onChange={(e) => setFormData({ ...formData, description: e.target.value })}
      />
      <button type="button" onClick={classifyTicket}>
        Get AI Suggestion
      </button>
      {aiSuggestion && (
        <div className="ai-suggestion">
          <p>Category: {aiSuggestion.category}</p>
          <p>Priority: {aiSuggestion.priority}</p>
          <p>Confidence: {(aiSuggestion.confidence * 100).toFixed(1)}%</p>
        </div>
      )}
      <button type="submit">Create Ticket</button>
    </form>
  );
};

export default CreateTicket;
```

## Backend Patterns

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const bcrypt = require('bcryptjs');

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role,
      department
    });

    await user.save();
    res.status(201).json({ message: 'User created successfully', user });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { userId } = req.params;
    const updates = req.body;

    if (updates.password) {
      updates.password = await bcrypt.hash(updates.password, 10);
    }

    const user = await User.findByIdAndUpdate(userId, updates, { new: true })
      .select('-password');
    
    res.json({ message: 'User updated successfully', user });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.deleteUser = async (req, res) => {
  try {
    const { userId } = req.params;
    await User.findByIdAndDelete(userId);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

exports.authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ message: 'Access token required' });
  }

  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ message: 'Invalid token' });
    }
    req.user = user;
    next();
  });
};

exports.requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};
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
    def __init__(self):
        self.vectorizer = TfidfVectorizer(max_features=1000)
        self.category_model = MultinomialNB()
        self.priority_model = MultinomialNB()
        self.trained = False
    
    def train(self, texts, categories, priorities):
        X = self.vectorizer.fit_transform(texts)
        self.category_model.fit(X, categories)
        self.priority_model.fit(X, priorities)
        self.trained = True
    
    def predict(self, text):
        if not self.trained:
            return {
                'category': 'general',
                'priority': 'medium',
                'confidence': 0.5
            }
        
        X = self.vectorizer.transform([text])
        category = self.category_model.predict(X)[0]
        priority = self.priority_model.predict(X)[0]
        confidence = max(self.category_model.predict_proba(X)[0])
        
        return {
            'category': category,
            'priority': priority,
            'confidence': float(confidence)
        }
    
    def save(self, path):
        os.makedirs(path, exist_ok=True)
        with open(f'{path}/vectorizer.pkl', 'wb') as f:
            pickle.dump(self.vectorizer, f)
        with open(f'{path}/category_model.pkl', 'wb') as f:
            pickle.dump(self.category_model, f)
        with open(f'{path}/priority_model.pkl', 'wb') as f:
            pickle.dump(self.priority_model, f)
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.ticket_classifier import TicketClassifier
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
import os

app = FastAPI()

ticket_classifier = TicketClassifier()
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()

class TicketInput(BaseModel):
    subject: str
    description: str

class RiskInput(BaseModel):
    userId: str
    failedLoginAttempts: int
    dataAccessPatterns: list[float]
    workHours: list[float]

class BurnoutInput(BaseModel):
    userId: str
    tasksCompleted: int
    averageTaskTime: float
    overtimeHours: float
    weeklyWorkHours: list[float]

@app.post("/ml/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    try:
        combined_text = f"{ticket.subject} {ticket.description}"
        result = ticket_classifier.predict(combined_text)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/predict-risk")
async def predict_risk(data: RiskInput):
    try:
        features = {
            'failed_logins': data.failedLoginAttempts,
            'access_patterns': data.dataAccessPatterns,
            'work_hours': data.workHours
        }
        result = risk_predictor.predict(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/detect-burnout")
async def detect_burnout(data: BurnoutInput):
    try:
        features = {
            'tasks_completed': data.tasksCompleted,
            'avg_task_time': data.averageTaskTime,
            'overtime_hours': data.overtimeHours,
            'weekly_hours': data.weeklyWorkHours
        }
        result = burnout_detector.predict(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
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
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  isActive: { type: Boolean, default: true }
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
  timeSpent: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

## Troubleshooting

### Common Issues

**JWT Token Expiration**
```javascript
// Add token refresh logic
const refreshToken = async () => {
  try {
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/api/auth/refresh`,
      { refreshToken: localStorage.getItem('refreshToken') }
    );
    localStorage.setItem('token', response.data.token);
    return response.data.token;
  } catch (error) {
    logout();
  }
};
```

**CORS Issues**
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

**ML Service Connection Errors**
```python
# ml-service/main.py
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**MongoDB Connection Issues**
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

## Performance Optimization

### Implement Caching

```javascript
// backend/middleware/cache.js
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 });

exports.cacheMiddleware = (duration) => {
  return (req, res, next) => {
    const key = req.originalUrl;
    const cachedResponse = cache.get(key);
    
    if (cachedResponse) {
      return res.json(cachedResponse);
    }
    
    res.originalJson = res.json;
    res.json = (data) => {
      cache.set(key, data, duration);
      res.originalJson(data);
    };
    next();
  };
};
```

### Database Indexing

```javascript
// backend/models/Task.js
taskSchema.index({ assignedTo: 1, status: 1 });
taskSchema.index({ createdAt: -1 });
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to assist developers in implementing, extending, and troubleshooting the system effectively.
