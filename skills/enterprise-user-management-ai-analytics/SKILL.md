---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement AI-powered user task management system"
  - "configure ticket classification and routing with ML"
  - "add burnout detection and risk prediction analytics"
  - "build admin dashboard with AI insights"
  - "integrate JWT authentication for user management"
  - "create Kanban board with time tracking"
  - "deploy user management system with FastAPI ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application combining React frontend, Node.js backend, and FastAPI ML service to provide intelligent user, task, and ticket management. The system uses AI/ML for automated ticket classification, risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Key Capabilities:**
- User authentication and role-based access control (JWT)
- Kanban task management with time tracking
- AI-powered support ticket classification and routing
- ML-based risk prediction and anomaly detection
- Burnout detection using workload analysis
- Predictive analytics for project delays
- Admin dashboard with audit logs and analytics

## Installation

### Prerequisites
```bash
# Node.js 14+ for backend/frontend
node --version

# Python 3.8+ for ML service
python --version

# MongoDB instance
```

### Clone and Setup
```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)
```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=30d
ML_SERVICE_URL=http://localhost:8000
EOF

# Start backend
npm start
# Runs at http://localhost:5000
```

### ML Service Setup (FastAPI)
```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup (React)
```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Runs at http://localhost:3000
```

## Project Structure

```
enterprise-user-management/
├── frontend/          # React application
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── services/  # API integration
│   │   └── utils/
│   └── package.json
├── backend/           # Node.js REST API
│   ├── controllers/
│   ├── models/
│   ├── routes/
│   ├── middleware/
│   └── server.js
└── ml-service/        # FastAPI ML service
    ├── models/
    ├── services/
    ├── main.py
    └── requirements.txt
```

## Backend API Usage

### Authentication Routes
```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const user = await User.create({
      username,
      email,
      password,
      role: role || 'user'
    });
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ success: true, token, user });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user || !(await user.matchPassword(password))) {
      return res.status(401).json({ success: false, error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ success: true, token, user });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### Task Management Routes
```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect, authorize } = require('../middleware/auth');

// Get user tasks
router.get('/', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'username email')
      .sort('-createdAt');
    
    res.json({ success: true, data: tasks });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Create task (admin only)
router.post('/', protect, authorize('admin'), async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      status: 'todo',
      createdBy: req.user.id
    });
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { status },
      { new: true, runValidators: true }
    );
    
    if (!task) {
      return res.status(404).json({ success: false, error: 'Task not found' });
    }
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### Ticket Management Routes
```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const axios = require('axios');
const { protect } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { subject, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { subject, description }
    );
    
    const ticket = await Ticket.create({
      subject,
      description,
      priority,
      category: mlResponse.data.category,
      suggestedDepartment: mlResponse.data.department,
      createdBy: req.user.id,
      status: 'open'
    });
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Get tickets
router.get('/', protect, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { createdBy: req.user.id };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .sort('-createdAt');
    
    res.json({ success: true, data: tickets });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI Main Application
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import numpy as np
from datetime import datetime
import joblib
import os

app = FastAPI(title="Enterprise ML Service")

# Models directory
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    department: str
    confidence: float

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    unusual_hours_access: int
    data_download_volume: float
    location_changes: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_completion_time: float
    overtime_hours: float
    workload_score: float

# Load or initialize models
try:
    ticket_classifier = joblib.load(f'{MODEL_PATH}/ticket_classifier.pkl')
    risk_predictor = joblib.load(f'{MODEL_PATH}/risk_predictor.pkl')
except:
    # Initialize with dummy models if not found
    ticket_classifier = None
    risk_predictor = None

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and route to appropriate department"""
    try:
        text = f"{request.subject} {request.description}".lower()
        
        # Simple rule-based classification (replace with actual ML model)
        if any(word in text for word in ['password', 'login', 'access', 'authentication']):
            category = 'Access & Security'
            department = 'IT Security'
        elif any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'Technical Issue'
            department = 'Engineering'
        elif any(word in text for word in ['payment', 'billing', 'invoice', 'refund']):
            category = 'Billing'
            department = 'Finance'
        elif any(word in text for word in ['feature', 'request', 'enhancement']):
            category = 'Feature Request'
            department = 'Product'
        else:
            category = 'General Support'
            department = 'Customer Support'
        
        return TicketClassificationResponse(
            category=category,
            department=department,
            confidence=0.85
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk level based on behavior patterns"""
    try:
        # Calculate risk score (0-100)
        risk_score = (
            request.failed_logins * 10 +
            request.unusual_hours_access * 5 +
            min(request.data_download_volume / 1000, 20) +
            request.location_changes * 8
        )
        
        risk_score = min(risk_score, 100)
        
        if risk_score >= 70:
            risk_level = 'high'
            action = 'Immediate review required'
        elif risk_score >= 40:
            risk_level = 'medium'
            action = 'Monitor closely'
        else:
            risk_level = 'low'
            action = 'Normal activity'
        
        return {
            'user_id': request.user_id,
            'risk_score': round(risk_score, 2),
            'risk_level': risk_level,
            'recommended_action': action,
            'timestamp': datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk based on workload metrics"""
    try:
        # Calculate burnout score
        completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
        
        burnout_score = (
            (1 - completion_rate) * 30 +
            min(request.avg_task_completion_time / 8, 1) * 25 +
            min(request.overtime_hours / 40, 1) * 25 +
            min(request.workload_score / 10, 1) * 20
        )
        
        if burnout_score >= 70:
            risk_level = 'high'
            recommendation = 'Reduce workload immediately, schedule check-in'
        elif burnout_score >= 40:
            risk_level = 'medium'
            recommendation = 'Monitor workload, consider task redistribution'
        else:
            risk_level = 'low'
            recommendation = 'Workload appears sustainable'
        
        return {
            'user_id': request.user_id,
            'burnout_score': round(burnout_score, 2),
            'risk_level': risk_level,
            'completion_rate': round(completion_rate * 100, 1),
            'recommendation': recommendation,
            'metrics': {
                'tasks_assigned': request.tasks_assigned,
                'tasks_completed': request.tasks_completed,
                'overtime_hours': request.overtime_hours
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
async def predict_project_delay(
    total_tasks: int,
    completed_tasks: int,
    days_elapsed: int,
    days_remaining: int,
    avg_completion_rate: float
):
    """Predict if project will be delayed based on current progress"""
    try:
        current_progress = completed_tasks / total_tasks
        expected_progress = days_elapsed / (days_elapsed + days_remaining)
        
        remaining_tasks = total_tasks - completed_tasks
        projected_days = remaining_tasks / max(avg_completion_rate, 0.1)
        
        delay_days = max(0, projected_days - days_remaining)
        
        if delay_days > 5:
            status = 'high_risk'
            message = f'Project likely delayed by {int(delay_days)} days'
        elif delay_days > 2:
            status = 'at_risk'
            message = 'Project at risk of minor delay'
        else:
            status = 'on_track'
            message = 'Project on track for completion'
        
        return {
            'status': status,
            'message': message,
            'current_progress': round(current_progress * 100, 1),
            'expected_progress': round(expected_progress * 100, 1),
            'delay_days': int(delay_days),
            'completion_rate': round(avg_completion_rate, 2)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Integration

### Authentication Service
```javascript
// frontend/src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

class AuthService {
  async login(email, password) {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('user', JSON.stringify(response.data));
    }
    
    return response.data;
  }
  
  async register(userData) {
    const response = await axios.post(`${API_URL}/api/auth/register`, userData);
    return response.data;
  }
  
  logout() {
    localStorage.removeItem('user');
  }
  
  getCurrentUser() {
    return JSON.parse(localStorage.getItem('user'));
  }
  
  getAuthHeader() {
    const user = this.getCurrentUser();
    if (user && user.token) {
      return { Authorization: `Bearer ${user.token}` };
    }
    return {};
  }
}

export default new AuthService();
```

### Task Management Component
```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import authService from '../services/authService';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks`, {
        headers: authService.getAuthHeader()
      });
      
      const categorized = {
        todo: response.data.data.filter(t => t.status === 'todo'),
        inProgress: response.data.data.filter(t => t.status === 'in-progress'),
        done: response.data.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };
  
  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: authService.getAuthHeader() }
      );
      
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };
  
  const TaskCard = ({ task }) => (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => updateTaskStatus(
            task._id,
            task.status === 'todo' ? 'in-progress' : 'done'
          )}>
            Move →
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

### AI Analytics Dashboard
```javascript
// frontend/src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;
  
  const analyzeBurnout = async (userId) => {
    try {
      const response = await axios.post(`${ML_API_URL}/analyze-burnout`, {
        user_id: userId,
        tasks_assigned: 25,
        tasks_completed: 18,
        avg_task_completion_time: 6.5,
        overtime_hours: 15,
        workload_score: 7.8
      });
      
      setAnalytics(prev => ({ ...prev, burnout: response.data }));
    } catch (error) {
      console.error('Burnout analysis error:', error);
    }
  };
  
  const predictRisk = async (userId) => {
    try {
      const response = await axios.post(`${ML_API_URL}/predict-risk`, {
        user_id: userId,
        failed_logins: 2,
        unusual_hours_access: 3,
        data_download_volume: 500,
        location_changes: 1
      });
      
      setAnalytics(prev => ({ ...prev, risk: response.data }));
    } catch (error) {
      console.error('Risk prediction error:', error);
    }
  };
  
  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics & Insights</h2>
      
      {analytics?.burnout && (
        <div className="analytics-card">
          <h3>Burnout Analysis</h3>
          <div className={`score score-${analytics.burnout.risk_level}`}>
            Score: {analytics.burnout.burnout_score}
          </div>
          <p><strong>Risk Level:</strong> {analytics.burnout.risk_level}</p>
          <p><strong>Completion Rate:</strong> {analytics.burnout.completion_rate}%</p>
          <p><strong>Recommendation:</strong> {analytics.burnout.recommendation}</p>
        </div>
      )}
      
      {analytics?.risk && (
        <div className="analytics-card">
          <h3>Security Risk Assessment</h3>
          <div className={`score score-${analytics.risk.risk_level}`}>
            Score: {analytics.risk.risk_score}
          </div>
          <p><strong>Risk Level:</strong> {analytics.risk.risk_level}</p>
          <p><strong>Action:</strong> {analytics.risk.recommended_action}</p>
        </div>
      )}
      
      <button onClick={() => analyzeBurnout('user123')}>
        Run Burnout Analysis
      </button>
      <button onClick={() => predictRisk('user123')}>
        Check Risk Level
      </button>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Database Models

### User Model
```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    match: [/^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/, 'Please add a valid email']
  },
  password: {
    type: String,
    required: true,
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Encrypt password before saving
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Match password
UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Model
```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: Date,
  timeTracked: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', TaskSchema);
```

## Common Patterns

### Protected Route Middleware
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
    return res.status(401).json({ success: false, error: 'Not authorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (error) {
    return res.status(401).json({ success: false, error: 'Not authorized' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: `User role ${req.user.role} is not authorized`
      });
    }
    next();
  };
};
```

### API Error Handling
```javascript
// backend/middleware/errorHandler.js
const errorHandler = (err, req, res, next) => {
  let error = { ...err };
  error.message = err.message;
  
  // Mongoose bad ObjectId
  if (err.name === 'CastError') {
    const message = 'Resource not found';
    error = { message, statusCode: 404 };
  }
  
  // Mongoose duplicate key
  if (err.code === 11000) {
    const message = 'Duplicate field value entered';
    error = { message, statusCode: 400 };
  }
  
  // Mongoose validation error
  if (err.name === 'ValidationError') {
    const message = Object.values(err.errors).map(val => val.message);
    error = { message, statusCode: 400 };
  }
  
  res.status(error.statusCode || 500).json({
    success: false,
    error: error.message || 'Server Error'
  });
};

module.exports = errorHandler;
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
NODE_ENV=development
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your-super-secret-jwt-key-change-in-production
JWT_EXPIRE=30d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=info
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5000
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENVIRONMENT=development
```

### Database Connection
```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

## Troubleshooting

### MongoDB Connection Issues
```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection string
echo $MONGODB_URI
```

### JWT Authentication Errors
```javascript
// Verify token is being sent correctly
const token = localStorage.getItem('user') ? 
  JSON.parse(localStorage.getItem('user')).token : null;

console.log('Token:', token);

// Check token expiration
const decoded = jwt.decode(token);
console.log('Token expires:', new Date(decoded.exp * 1000));
```

### CORS Issues
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Not Responding
```bash
# Check if service is running
curl http://localhost:8000/health

# View logs
uvicorn main:app --reload --log-level debug

# Check Python dependencies
pip list | grep -E "(fastapi|uvicorn|scikit-learn)"
```

### Task Status Update Fails
```javascript
// Ensure valid status values
const validStatuses = ['todo', 'in-progress', 'done'];
if (!validStatuses.includes(newStatus)) {
  console.error('Invalid status:', newStatus);
}

// Check
