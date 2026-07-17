---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "build user management with task tracking"
  - "implement AI-powered ticket classification"
  - "create admin dashboard with analytics"
  - "add burnout detection to user system"
  - "configure user management with ML service"
  - "deploy enterprise user management app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with AI-powered insights. The system provides admin controls, user dashboards with Kanban boards, time tracking, support ticket management, and ML-based analytics including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Architecture:**
- Frontend: React.js
- Backend: Node.js with REST APIs
- ML Service: FastAPI with scikit-learn and River
- Database: MongoDB
- Authentication: JWT

## Installation

### Prerequisites

```bash
# Node.js 14+ and Python 3.8+ required
node --version
python --version
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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
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
```

## Project Structure

```
Enterprise-User-Management-System-with-AI-Analytics/
├── frontend/              # React application
│   ├── src/
│   │   ├── components/   # React components
│   │   ├── pages/        # Page components
│   │   ├── services/     # API services
│   │   ├── context/      # React context
│   │   └── utils/        # Helper functions
├── backend/              # Node.js server
│   ├── models/          # MongoDB schemas
│   ├── routes/          # API routes
│   ├── controllers/     # Route controllers
│   ├── middleware/      # Auth & validation
│   └── utils/           # Helper functions
└── ml-service/          # FastAPI ML service
    ├── models/          # ML models
    ├── services/        # ML logic
    └── api/             # API endpoints
```

## Backend API

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = await User.create({
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });

    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;

    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### User Management

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { protect, authorize } = require('../middleware/auth');

// Get all users (admin only)
router.get('/', protect, authorize('admin'), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, data: users });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user by ID
router.get('/:id', protect, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update user
router.put('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Delete user
router.delete('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json({ success: true, message: 'User deleted' });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
  },
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
  dueDate: {
    type: Date
  },
  timeSpent: {
    type: Number,
    default: 0 // in minutes
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  }
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect } = require('../middleware/auth');

// Get all tasks for user
router.get('/my-tasks', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    res.json({ success: true, data: tasks });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task
router.post('/', protect, async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status },
      { new: true }
    );
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Track time on task
router.patch('/:id/time', protect, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findById(req.params.id);
    task.timeSpent += timeSpent;
    await task.save();
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Support Tickets

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
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'urgent'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical']
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
    type: Object
  }
}, { timestamps: true });

module.exports = mongoose.model('Ticket', ticketSchema);
```

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { protect } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, category } = req.body;
    
    // Call ML service for classification
    let aiClassification = null;
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description, category }
      );
      aiClassification = mlResponse.data;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }

    const ticket = await Ticket.create({
      title,
      description,
      category,
      priority: aiClassification?.priority || 'medium',
      createdBy: req.user.id,
      aiClassification
    });

    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', protect, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .populate('createdBy', 'name email')
      .populate('assignedTo', 'name email')
      .sort('-createdAt');
    res.json({ success: true, data: tickets });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import numpy as np
from datetime import datetime
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")
os.makedirs(MODEL_PATH, exist_ok=True)

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    category: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    task_completion_rate: float
    avg_response_time: float
    ticket_count: int
    late_submissions: int

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_hours_per_day: float
    overtime_hours: float

@app.get("/")
def read_root():
    return {"status": "ML Service Running", "version": "1.0.0"}

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

### Ticket Classification

```python
# ml-service/services/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import joblib
import os

class TicketClassifier:
    def __init__(self, model_path="./models"):
        self.model_path = model_path
        self.model = None
        self.load_or_create_model()
    
    def load_or_create_model(self):
        model_file = os.path.join(self.model_path, "ticket_classifier.pkl")
        if os.path.exists(model_file):
            self.model = joblib.load(model_file)
        else:
            # Create new model
            self.model = Pipeline([
                ('tfidf', TfidfVectorizer(max_features=100)),
                ('clf', MultinomialNB())
            ])
    
    def classify(self, title: str, description: str, category: str):
        text = f"{title} {description} {category}"
        
        # Rule-based classification for priority
        priority = "medium"
        if any(word in text.lower() for word in ['urgent', 'critical', 'emergency', 'asap']):
            priority = "critical"
        elif any(word in text.lower() for word in ['important', 'high', 'priority']):
            priority = "high"
        elif any(word in text.lower() for word in ['low', 'minor', 'simple']):
            priority = "low"
        
        # Route to appropriate team
        routing = self._route_ticket(category, text)
        
        return {
            "priority": priority,
            "routing": routing,
            "confidence": 0.85
        }
    
    def _route_ticket(self, category: str, text: str):
        if category == "technical":
            if "database" in text.lower() or "sql" in text.lower():
                return "database_team"
            elif "frontend" in text.lower() or "ui" in text.lower():
                return "frontend_team"
            else:
                return "backend_team"
        elif category == "billing":
            return "finance_team"
        else:
            return "support_team"

# Add to main.py
from services.ticket_classifier import TicketClassifier

ticket_classifier = TicketClassifier(MODEL_PATH)

@app.post("/classify-ticket")
def classify_ticket(request: TicketClassificationRequest):
    try:
        result = ticket_classifier.classify(
            request.title,
            request.description,
            request.category
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Risk Prediction

```python
# ml-service/services/risk_predictor.py
import numpy as np
from sklearn.ensemble import RandomForestClassifier

class RiskPredictor:
    def __init__(self):
        self.model = RandomForestClassifier(n_estimators=100, random_state=42)
        self.is_trained = False
    
    def predict_risk(self, features: dict):
        """
        Predict risk level based on user metrics
        features: {
            'task_completion_rate': float,
            'avg_response_time': float,
            'ticket_count': int,
            'late_submissions': int
        }
        """
        risk_score = 0
        risk_factors = []
        
        # Calculate risk score
        if features['task_completion_rate'] < 0.7:
            risk_score += 30
            risk_factors.append("Low task completion rate")
        
        if features['avg_response_time'] > 48:  # hours
            risk_score += 25
            risk_factors.append("Slow response time")
        
        if features['ticket_count'] > 10:
            risk_score += 20
            risk_factors.append("High ticket count")
        
        if features['late_submissions'] > 3:
            risk_score += 25
            risk_factors.append("Multiple late submissions")
        
        # Determine risk level
        if risk_score >= 70:
            risk_level = "high"
        elif risk_score >= 40:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return {
            "risk_level": risk_level,
            "risk_score": risk_score,
            "risk_factors": risk_factors,
            "recommendations": self._get_recommendations(risk_level, risk_factors)
        }
    
    def _get_recommendations(self, risk_level: str, risk_factors: list):
        recommendations = []
        
        if "Low task completion rate" in risk_factors:
            recommendations.append("Review workload and redistribute tasks")
        
        if "Slow response time" in risk_factors:
            recommendations.append("Check for blockers or provide additional support")
        
        if "High ticket count" in risk_factors:
            recommendations.append("Investigate recurring issues")
        
        if "Multiple late submissions" in risk_factors:
            recommendations.append("Schedule one-on-one to identify challenges")
        
        return recommendations

# Add to main.py
from services.risk_predictor import RiskPredictor

risk_predictor = RiskPredictor()

@app.post("/predict-risk")
def predict_risk(request: RiskPredictionRequest):
    try:
        features = {
            'task_completion_rate': request.task_completion_rate,
            'avg_response_time': request.avg_response_time,
            'ticket_count': request.ticket_count,
            'late_submissions': request.late_submissions
        }
        result = risk_predictor.predict_risk(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/services/burnout_detector.py
class BurnoutDetector:
    def detect_burnout(self, metrics: dict):
        """
        Detect burnout risk based on workload metrics
        """
        burnout_score = 0
        warning_signs = []
        
        # Task overload
        task_ratio = metrics['tasks_completed'] / max(metrics['tasks_assigned'], 1)
        if metrics['tasks_assigned'] > 15:
            burnout_score += 25
            warning_signs.append("High task load")
        
        if task_ratio < 0.6:
            burnout_score += 20
            warning_signs.append("Struggling to complete tasks")
        
        # Working hours
        if metrics['avg_hours_per_day'] > 10:
            burnout_score += 30
            warning_signs.append("Excessive working hours")
        
        if metrics['overtime_hours'] > 20:
            burnout_score += 25
            warning_signs.append("High overtime")
        
        # Determine burnout level
        if burnout_score >= 70:
            burnout_level = "high"
            action = "Immediate intervention required"
        elif burnout_score >= 40:
            burnout_level = "moderate"
            action = "Monitor closely and reduce workload"
        else:
            burnout_level = "low"
            action = "Continue monitoring"
        
        return {
            "burnout_level": burnout_level,
            "burnout_score": burnout_score,
            "warning_signs": warning_signs,
            "recommended_action": action,
            "suggestions": self._get_suggestions(warning_signs)
        }
    
    def _get_suggestions(self, warning_signs: list):
        suggestions = []
        
        if "High task load" in warning_signs:
            suggestions.append("Redistribute tasks to balance workload")
        
        if "Excessive working hours" in warning_signs:
            suggestions.append("Enforce work-life balance policies")
        
        if "High overtime" in warning_signs:
            suggestions.append("Review project timelines and add resources")
        
        return suggestions

# Add to main.py
from services.burnout_detector import BurnoutDetector

burnout_detector = BurnoutDetector()

@app.post("/detect-burnout")
def detect_burnout(request: BurnoutDetectionRequest):
    try:
        metrics = {
            'tasks_assigned': request.tasks_assigned,
            'tasks_completed': request.tasks_completed,
            'avg_hours_per_day': request.avg_hours_per_day,
            'overtime_hours': request.overtime_hours
        }
        result = burnout_detector.detect_burnout(metrics)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Integration

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

// Create axios instance
const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Auth API
export const authAPI = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData),
  getCurrentUser: () => api.get('/auth/me')
};

// User API
export const userAPI = {
  getAll: () => api.get('/users'),
  getById: (id) => api.get(`/users/${id}`),
  update: (id, data) => api.put(`/users/${id}`, data),
  delete: (id) => api.delete(`/users/${id}`)
};

// Task API
export const taskAPI = {
  getMyTasks: () => api.get('/tasks/my-tasks'),
  create: (taskData) => api.post('/tasks', taskData),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  trackTime: (id, timeSpent) => api.patch(`/tasks/${id}/time`, { timeSpent })
};

// Ticket API
export const ticketAPI = {
  create: (ticketData) => api.post('/tickets', ticketData),
  getMyTickets: () => api.get('/tickets/my-tickets'),
  getAll: () => api.get('/tickets')
};

// ML API
export const mlAPI = {
  classifyTicket: (ticketData) => 
    axios.post(`${ML_URL}/classify-ticket`, ticketData),
  predictRisk: (userData) => 
    axios.post(`${ML_URL}/predict-risk`, userData),
  detectBurnout: (metrics) => 
    axios.post(`${ML_URL}/detect-burnout`, metrics)
};

export default api;
```

### Task Component with Kanban

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const response = await taskAPI.getMyTasks();
      const grouped = {
        todo: [],
        'in-progress': [],
        done: []
      };
      
      response.data.data.forEach(task => {
        grouped[task.status].push(task);
      });
      
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Error loading tasks:', error);
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskAPI.updateStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const renderTask = (task) => (
    <div key={task._id} className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority priority-${task.priority}`}>
          {task.priority}
        </span>
        <span className="time-spent">{task.timeSpent}m</span>
      </div>
      <div className="task-actions">
        {task.status === 'todo' && (
          <button onClick={() => handleStatusChange(task._id, 'in-progress')}>
            Start
          </button>
        )}
        {task.status === 'in-progress' && (
          <>
            <button onClick={() => handleStatusChange(task._id, 'todo')}>
              Back
            </button>
            <button onClick={() => handleStatusChange(task._id, 'done')}>
              Complete
            </button>
          </>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="task-board">
      <div className="column">
        <h3>To Do ({tasks.todo.length})</h3>
        <div className="task-list">
          {tasks.todo.map(renderTask)}
        </div>
      </div>
      
      <div className="column">
        <h3>In Progress ({tasks['in-progress'].length})</h3>
        <div className="task-list">
          {tasks['in-progress'].map(renderTask)}
        </div>
      </div>
      
      <div className="column">
        <h3>Done ({tasks.done.length})</h3>
        <div className="task-list">
          {tasks.done.map(renderTask)}
        </div>
      </div>
    </div>
  );
};

export default TaskBoard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { userAPI, mlAPI } from '../services/api';
import './AdminDashboard.css';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState({});
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const usersResponse = await userAPI.getAll();
      setUsers(usersResponse.data.data);
      
      // Load analytics for each user
      const analyticsData = {};
      for (const user of usersResponse.data.data) {
        const riskData = await mlAPI.predictRisk({
          user_id: user._id,
          task_completion_rate: user.taskCompletionRate || 0.8,
          avg_response_time: user.avgResponseTime || 24,
          ticket_count: user.ticketCount || 0,
          late_submissions: user.lateSubmissions || 0
        });
