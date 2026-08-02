---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, ticket routing, risk detection, and burnout prediction
triggers:
  - "build a user management system with AI analytics"
  - "implement AI-powered ticket classification and routing"
  - "create role-based access control with JWT authentication"
  - "add burnout detection and risk prediction to user management"
  - "set up Kanban board with time tracking for tasks"
  - "integrate AI analytics dashboard for enterprise users"
  - "build admin panel with anomaly detection"
  - "implement predictive project insights with machine learning"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform combining React.js, Node.js, and FastAPI ML services to provide intelligent user administration, task management, and AI-driven analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

This system provides:
- **User Management**: JWT-authenticated role-based access control (admin/user roles)
- **Task Management**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Dashboards**: Admin analytics and user performance dashboards
- **Audit Logs**: Activity tracking and suspicious behavior alerts

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or Atlas)

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install
```

Create `backend/.env`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:
```bash
npm start
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt
```

Create `ml-service/.env`:
```env
MODEL_PATH=./models
LOG_LEVEL=info
BACKEND_URL=http://localhost:5000
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install
```

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Key Backend API Patterns

### Authentication (Node.js/Express)

```javascript
// backend/middleware/auth.js
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

### User Management Routes

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authMiddleware, adminOnly } = require('../middleware/auth');

// Get all users (admin only)
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create user (admin only)
router.post('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const { username, email, password, role, department } = req.body;
    const user = new User({ username, email, password, role, department });
    await user.save();
    res.status(201).json({ message: 'User created', userId: user._id });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update user
router.put('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const updates = req.body;
    const user = await User.findByIdAndUpdate(req.params.id, updates, { new: true }).select('-password');
    res.json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  timeSpent: { type: Number, default: 0 }, // in seconds
  dueDate: Date,
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get user tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'username')
      .sort('-createdAt');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status (Kanban)
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { status },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Track time
router.post('/:id/track-time', authMiddleware, async (req, res) => {
  try {
    const { seconds } = req.body;
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { $inc: { timeSpent: seconds } },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service API (FastAPI)

### Ticket Classification

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import pickle
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB

app = FastAPI()

# Load or initialize models
try:
    with open('models/ticket_classifier.pkl', 'rb') as f:
        ticket_model = pickle.load(f)
    with open('models/vectorizer.pkl', 'rb') as f:
        vectorizer = pickle.load(f)
except:
    vectorizer = TfidfVectorizer(max_features=500)
    ticket_model = MultinomialNB()

class TicketRequest(BaseModel):
    title: str
    description: str
    priority: Optional[str] = "medium"

class TicketResponse(BaseModel):
    category: str
    department: str
    confidence: float

@app.post("/api/classify-ticket", response_model=TicketResponse)
async def classify_ticket(ticket: TicketRequest):
    try:
        text = f"{ticket.title} {ticket.description}"
        features = vectorizer.transform([text])
        prediction = ticket_model.predict(features)[0]
        confidence = ticket_model.predict_proba(features).max()
        
        # Map categories to departments
        dept_mapping = {
            "technical": "IT Support",
            "billing": "Finance",
            "hr": "Human Resources",
            "general": "General Support"
        }
        
        return TicketResponse(
            category=prediction,
            department=dept_mapping.get(prediction, "General Support"),
            confidence=float(confidence)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Risk Detection

```python
# ml-service/risk_detection.py
from pydantic import BaseModel
from typing import List
import numpy as np

class UserActivity(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    tasksCompleted: int
    tasksOverdue: int
    avgTaskCompletionTime: float
    lastLoginHour: int
    accountAge: int  # days

class RiskScore(BaseModel):
    userId: str
    riskLevel: str
    score: float
    factors: List[str]

@app.post("/api/analyze-risk", response_model=RiskScore)
async def analyze_risk(activity: UserActivity):
    risk_score = 0.0
    factors = []
    
    # Failed login analysis
    if activity.failedLogins > 3:
        risk_score += 0.3
        factors.append("High failed login attempts")
    
    # Task performance
    if activity.tasksOverdue > activity.tasksCompleted * 0.3:
        risk_score += 0.2
        factors.append("High overdue task ratio")
    
    # Unusual login times
    if activity.lastLoginHour < 6 or activity.lastLoginHour > 22:
        risk_score += 0.15
        factors.append("Unusual login time")
    
    # Performance decline
    if activity.avgTaskCompletionTime > 72:  # hours
        risk_score += 0.2
        factors.append("Slow task completion")
    
    # New account behavior
    if activity.accountAge < 7 and activity.loginAttempts > 10:
        risk_score += 0.15
        factors.append("Unusual new account activity")
    
    # Determine risk level
    if risk_score < 0.3:
        level = "low"
    elif risk_score < 0.6:
        level = "medium"
    else:
        level = "high"
    
    return RiskScore(
        userId=activity.userId,
        riskLevel=level,
        score=min(risk_score, 1.0),
        factors=factors
    )
```

### Burnout Detection

```python
# ml-service/burnout_detection.py
class WorkloadMetrics(BaseModel):
    userId: str
    weeklyHours: float
    weekendWork: int
    taskCount: int
    avgTaskDuration: float
    missedDeadlines: int
    lastBreakDays: int

class BurnoutAnalysis(BaseModel):
    userId: str
    burnoutRisk: str
    score: float
    recommendations: List[str]

@app.post("/api/detect-burnout", response_model=BurnoutAnalysis)
async def detect_burnout(metrics: WorkloadMetrics):
    burnout_score = 0.0
    recommendations = []
    
    # Weekly hours analysis
    if metrics.weeklyHours > 50:
        burnout_score += 0.25
        recommendations.append("Reduce weekly working hours")
    
    # Weekend work
    if metrics.weekendWork > 2:
        burnout_score += 0.2
        recommendations.append("Avoid weekend work")
    
    # Task overload
    if metrics.taskCount > 15:
        burnout_score += 0.2
        recommendations.append("Redistribute tasks")
    
    # Missed deadlines
    if metrics.missedDeadlines > 3:
        burnout_score += 0.2
        recommendations.append("Review task priorities")
    
    # Break time
    if metrics.lastBreakDays > 60:
        burnout_score += 0.15
        recommendations.append("Schedule time off")
    
    # Determine risk
    if burnout_score < 0.3:
        risk = "low"
    elif burnout_score < 0.6:
        risk = "moderate"
    else:
        risk = "high"
    
    return BurnoutAnalysis(
        userId=metrics.userId,
        burnoutRisk=risk,
        score=min(burnout_score, 1.0),
        recommendations=recommendations
    )
```

## Frontend Integration (React)

### API Client Setup

```javascript
// frontend/src/api/client.js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
});

const mlClient = axios.create({
  baseURL: process.env.REACT_APP_ML_URL,
});

// Add auth token to requests
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export { apiClient, mlClient };
```

### Task Kanban Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { apiClient } from '../api/client';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const { data } = await apiClient.get('/tasks/my-tasks');
      const grouped = data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await apiClient.patch(`/tasks/${taskId}/status`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('-', ' ').toUpperCase()}</h3>
          {tasks[status]?.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status} 
                onChange={(e) => moveTask(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in-progress">In Progress</option>
                <option value="done">Done</option>
              </select>
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
// frontend/src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import { apiClient, mlClient } from '../api/client';

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const { data: userActivity } = await apiClient.get('/users/my-activity');
      
      // Get risk analysis
      const { data: riskData } = await mlClient.post('/api/analyze-risk', userActivity);
      
      // Get burnout analysis
      const { data: burnoutData } = await mlClient.post('/api/detect-burnout', userActivity);
      
      setAnalytics({ risk: riskData, burnout: burnoutData });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="risk-card">
        <h3>Risk Assessment</h3>
        <p className={`risk-${analytics.risk.riskLevel}`}>
          Risk Level: {analytics.risk.riskLevel.toUpperCase()}
        </p>
        <p>Score: {(analytics.risk.score * 100).toFixed(1)}%</p>
        <ul>
          {analytics.risk.factors.map((factor, i) => (
            <li key={i}>{factor}</li>
          ))}
        </ul>
      </div>
      
      <div className="burnout-card">
        <h3>Burnout Analysis</h3>
        <p className={`burnout-${analytics.burnout.burnoutRisk}`}>
          Burnout Risk: {analytics.burnout.burnoutRisk.toUpperCase()}
        </p>
        <p>Score: {(analytics.burnout.score * 100).toFixed(1)}%</p>
        <h4>Recommendations:</h4>
        <ul>
          {analytics.burnout.recommendations.map((rec, i) => (
            <li key={i}>{rec}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Configuration

### Database Models

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
  isActive: { type: Boolean, default: true },
  lastLogin: Date,
  loginAttempts: { type: Number, default: 0 },
  accountLocked: { type: Boolean, default: false },
}, { timestamps: true });

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(password) {
  return bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Ticket Model with AI Integration

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: String,
  department: String,
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassified: { type: Boolean, default: false },
  aiConfidence: Number,
}, { timestamps: true });

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### AI-Powered Ticket Creation

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const axios = require('axios');
const { authMiddleware } = require('../middleware/auth');

router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Get AI classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/api/classify-ticket`, {
      title,
      description,
      priority
    });
    
    const { category, department, confidence } = mlResponse.data;
    
    const ticket = new Ticket({
      title,
      description,
      priority,
      category,
      department,
      createdBy: req.user.id,
      aiClassified: true,
      aiConfidence: confidence,
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Scheduled Risk Analysis

```javascript
// backend/jobs/riskAnalysis.js
const cron = require('node-cron');
const User = require('../models/User');
const Task = require('../models/Task');
const axios = require('axios');

// Run daily at midnight
cron.schedule('0 0 * * *', async () => {
  console.log('Running daily risk analysis...');
  
  const users = await User.find({ isActive: true });
  
  for (const user of users) {
    const tasks = await Task.find({ assignedTo: user._id });
    
    const activity = {
      userId: user._id.toString(),
      loginAttempts: user.loginAttempts,
      failedLogins: user.failedLoginCount || 0,
      tasksCompleted: tasks.filter(t => t.status === 'done').length,
      tasksOverdue: tasks.filter(t => t.dueDate < new Date() && t.status !== 'done').length,
      avgTaskCompletionTime: calculateAvgTime(tasks),
      lastLoginHour: user.lastLogin ? user.lastLogin.getHours() : 12,
      accountAge: Math.floor((Date.now() - user.createdAt) / (1000 * 60 * 60 * 24)),
    };
    
    try {
      const { data } = await axios.post(`${process.env.ML_SERVICE_URL}/api/analyze-risk`, activity);
      
      if (data.riskLevel === 'high') {
        // Send alert to admin
        console.log(`High risk detected for user ${user.username}`);
        // Implement notification logic
      }
    } catch (error) {
      console.error(`Risk analysis failed for ${user.username}:`, error.message);
    }
  }
});

function calculateAvgTime(tasks) {
  const completed = tasks.filter(t => t.status === 'done' && t.updatedAt && t.createdAt);
  if (completed.length === 0) return 0;
  
  const totalHours = completed.reduce((sum, task) => {
    const hours = (task.updatedAt - task.createdAt) / (1000 * 60 * 60);
    return sum + hours;
  }, 0);
  
  return totalHours / completed.length;
}

module.exports = {};
```

## Troubleshooting

### ML Service Connection Issues

If the backend cannot connect to the ML service:

```javascript
// backend/utils/mlClient.js
const axios = require('axios');

const mlClient = axios.create({
  baseURL: process.env.ML_SERVICE_URL,
  timeout: 5000,
});

// Fallback classification if ML service is down
const fallbackClassification = (title, description) => {
  const text = `${title} ${description}`.toLowerCase();
  
  if (text.includes('login') || text.includes('password')) {
    return { category: 'technical', department: 'IT Support', confidence: 0.6 };
  }
  if (text.includes('payment') || text.includes('invoice')) {
    return { category: 'billing', department: 'Finance', confidence: 0.6 };
  }
  if (text.includes('leave') || text.includes('salary')) {
    return { category: 'hr', department: 'Human Resources', confidence: 0.6 };
  }
  
  return { category: 'general', department: 'General Support', confidence: 0.5 };
};

mlClient.interceptors.response.use(
  response => response,
  error => {
    console.warn('ML service unavailable, using fallback');
    return { data: fallbackClassification('', '') };
  }
);

module.exports = { mlClient, fallbackClassification };
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
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected, attempting to reconnect...');
  setTimeout(connectDB, 5000);
});

module.exports = connectDB;
```

### JWT Token Expiration Handling

```javascript
// frontend/src/api/client.js
apiClient.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### ML Model Training Script

```python
# ml-service/train_models.py
import pickle
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import os

# Sample training data
training_data = [
    ("Cannot login to system", "technical"),
    ("Password reset required", "technical"),
    ("Invoice not received", "billing"),
    ("Payment pending", "billing"),
    ("Leave approval needed", "hr"),
    ("Salary query", "hr"),
]

texts, labels = zip(*training_data)

vectorizer = TfidfVectorizer(max_features=500)
X = vectorizer.fit_transform(texts)

model = MultinomialNB()
model.fit(X, labels)

# Save models
os.makedirs('models', exist_ok=True)
with open('models/ticket_classifier.pkl', 'wb') as f:
    pickle.dump(model, f)
with open('models/vectorizer.pkl', 'wb') as f:
    pickle.dump(vectorizer, f)

print("Models trained and saved successfully")
```

Run training:
```bash
cd ml-service
python train_models.py
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to help developers implement user management, task tracking, AI-powered analytics, and intelligent ticket routing.
