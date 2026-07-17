---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with ML"
  - "build task tracking system with AI insights"
  - "integrate intelligent ticket routing"
  - "add burnout detection and risk prediction"
  - "configure enterprise user management with AI"
  - "deploy user management system with analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system that combines user administration, task tracking, and support ticket management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

This system provides a centralized platform for enterprise user management with three main components:

1. **Backend API** (Node.js): User authentication, CRUD operations, task management, ticket handling
2. **ML Service** (FastAPI): AI-powered analytics, predictions, and intelligent routing
3. **Frontend** (React): Admin dashboard, user dashboard, Kanban boards, analytics visualization

Key capabilities:
- JWT-based authentication and role-based access control
- Task assignment with Kanban workflow (To Do → In Progress → Done)
- Time tracking and performance monitoring
- AI-based ticket classification and smart routing
- Risk prediction and anomaly detection
- Burnout analysis based on workload patterns
- Predictive project delay insights

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

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
MONGODB_URI=mongodb://localhost:27017/enterprise_users
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
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
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
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

## Backend API Usage

### Authentication

```javascript
// backend/controllers/authController.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register new user
exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = await User.create({
      name,
      email,
      password,
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
};

// Login
exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await user.comparePassword(password);
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
};
```

### User Management

```javascript
// backend/controllers/userController.js
const User = require('../models/User');

// Get all users (admin only)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, count: users.length, data: users });
  } catch (error) {
    res.status(500).json({ message: error.message });
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
    
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ message: error.message });
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
    
    res.json({ success: true, message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### Task Management

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Create task
exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, deadline } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      createdBy: req.user.id,
      priority: priority || 'medium',
      status: 'todo',
      deadline
    });
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Get user tasks
exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### Ticket Management

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

// Create ticket with AI classification
exports.createTicket = async (req, res) => {
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
      priority: priority || 'medium',
      createdBy: req.user.id,
      category: mlResponse.data.category,
      suggestedDepartment: mlResponse.data.department,
      status: 'open'
    });
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Get all tickets
exports.getTickets = async (req, res) => {
  try {
    const { status, priority, category } = req.query;
    const filter = {};
    
    if (status) filter.status = status;
    if (priority) filter.priority = priority;
    if (category) filter.category = category;
    
    const tickets = await Ticket.find(filter)
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, count: tickets.length, data: tickets });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

## ML Service API Usage

### AI-Powered Ticket Classification

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

app = FastAPI()

class TicketRequest(BaseModel):
    subject: str
    description: str

class TicketResponse(BaseModel):
    category: str
    department: str
    confidence: float

# Load or train classification model
def load_ticket_classifier():
    model_path = os.getenv('MODEL_PATH', './models')
    try:
        vectorizer = joblib.load(f'{model_path}/ticket_vectorizer.pkl')
        classifier = joblib.load(f'{model_path}/ticket_classifier.pkl')
    except FileNotFoundError:
        # Initialize with default model
        vectorizer = TfidfVectorizer(max_features=1000)
        classifier = MultinomialNB()
    return vectorizer, classifier

vectorizer, classifier = load_ticket_classifier()

@app.post("/classify-ticket", response_model=TicketResponse)
async def classify_ticket(ticket: TicketRequest):
    try:
        # Combine subject and description
        text = f"{ticket.subject} {ticket.description}"
        
        # Transform and predict
        features = vectorizer.transform([text])
        category = classifier.predict(features)[0]
        confidence = max(classifier.predict_proba(features)[0])
        
        # Map category to department
        department_mapping = {
            'technical': 'IT Support',
            'billing': 'Finance',
            'general': 'Customer Service',
            'security': 'Security Team'
        }
        
        return TicketResponse(
            category=category,
            department=department_mapping.get(category, 'General'),
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
    user_id: str
    login_attempts: int
    failed_logins: int
    unusual_hours: int
    data_access_volume: int
    permission_changes: int

class RiskScore(BaseModel):
    user_id: str
    risk_level: str
    score: float
    factors: List[str]

@app.post("/analyze-risk", response_model=RiskScore)
async def analyze_risk(activity: UserActivity):
    try:
        risk_factors = []
        risk_score = 0.0
        
        # Failed login analysis
        if activity.failed_logins > 3:
            risk_score += 0.3
            risk_factors.append("Multiple failed login attempts")
        
        # Unusual hours activity
        if activity.unusual_hours > 5:
            risk_score += 0.2
            risk_factors.append("Activity during unusual hours")
        
        # High data access
        if activity.data_access_volume > 1000:
            risk_score += 0.25
            risk_factors.append("Unusually high data access")
        
        # Permission changes
        if activity.permission_changes > 2:
            risk_score += 0.25
            risk_factors.append("Multiple permission changes")
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "high"
        elif risk_score >= 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskScore(
            user_id=activity.user_id,
            risk_level=risk_level,
            score=risk_score,
            factors=risk_factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/burnout_detection.py
from datetime import datetime, timedelta

class WorkloadData(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    average_task_duration: float
    overtime_hours: float
    days_without_break: int

class BurnoutAnalysis(BaseModel):
    user_id: str
    burnout_risk: str
    score: float
    recommendations: List[str]

@app.post("/detect-burnout", response_model=BurnoutAnalysis)
async def detect_burnout(workload: WorkloadData):
    try:
        burnout_score = 0.0
        recommendations = []
        
        # Task overload
        task_ratio = workload.tasks_assigned / max(workload.tasks_completed, 1)
        if task_ratio > 2:
            burnout_score += 0.3
            recommendations.append("Reduce task assignment load")
        
        # Long task durations
        if workload.average_task_duration > 8:
            burnout_score += 0.2
            recommendations.append("Break down complex tasks")
        
        # Overtime analysis
        if workload.overtime_hours > 10:
            burnout_score += 0.3
            recommendations.append("Limit overtime hours")
        
        # Days without break
        if workload.days_without_break > 14:
            burnout_score += 0.2
            recommendations.append("Schedule mandatory time off")
        
        # Determine risk level
        if burnout_score >= 0.6:
            risk = "high"
        elif burnout_score >= 0.3:
            risk = "medium"
        else:
            risk = "low"
        
        return BurnoutAnalysis(
            user_id=workload.user_id,
            burnout_risk=risk,
            score=burnout_score,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Predictive Analytics

```python
# ml-service/predictive_analytics.py
from river import linear_model, preprocessing, compose

class ProjectData(BaseModel):
    project_id: str
    tasks_total: int
    tasks_completed: int
    days_elapsed: int
    team_size: int
    complexity_score: float

class DelayPrediction(BaseModel):
    project_id: str
    delay_probability: float
    estimated_delay_days: int
    risk_factors: List[str]

# Initialize online learning model
delay_model = compose.Pipeline(
    preprocessing.StandardScaler(),
    linear_model.LogisticRegression()
)

@app.post("/predict-delay", response_model=DelayPrediction)
async def predict_delay(project: ProjectData):
    try:
        # Calculate features
        completion_rate = project.tasks_completed / max(project.tasks_total, 1)
        velocity = project.tasks_completed / max(project.days_elapsed, 1)
        
        risk_factors = []
        
        # Low completion rate
        if completion_rate < 0.3 and project.days_elapsed > 10:
            risk_factors.append("Low completion rate for elapsed time")
        
        # Slow velocity
        expected_velocity = project.tasks_total / (project.days_elapsed * 2)
        if velocity < expected_velocity:
            risk_factors.append("Below expected velocity")
        
        # High complexity with small team
        if project.complexity_score > 0.7 and project.team_size < 3:
            risk_factors.append("High complexity with limited resources")
        
        # Calculate delay probability
        delay_prob = 1 - completion_rate
        if velocity < expected_velocity:
            delay_prob += 0.2
        if project.complexity_score > 0.7:
            delay_prob += 0.15
        
        delay_prob = min(delay_prob, 1.0)
        
        # Estimate delay days
        remaining_tasks = project.tasks_total - project.tasks_completed
        estimated_days = int(remaining_tasks / max(velocity, 0.1))
        estimated_delay = max(0, estimated_days - project.days_elapsed)
        
        return DelayPrediction(
            project_id=project.project_id,
            delay_probability=float(delay_prob),
            estimated_delay_days=estimated_delay,
            risk_factors=risk_factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Usage

### React Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/auth/me`);
      setUser(response.data.data);
    } catch (error) {
      localStorage.removeItem('token');
      setToken(null);
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Dashboard Component

```javascript
// frontend/src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks/my-tasks`);
      const taskData = response.data.data;
      
      setTasks({
        todo: taskData.filter(t => t.status === 'todo'),
        inProgress: taskData.filter(t => t.status === 'in_progress'),
        done: taskData.filter(t => t.status === 'done')
      });
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/api/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
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
        {task.status === 'todo' && (
          <button onClick={() => updateTaskStatus(task._id, 'in_progress')}>
            Start
          </button>
        )}
        {task.status === 'in_progress' && (
          <button onClick={() => updateTaskStatus(task._id, 'done')}>
            Complete
          </button>
        )}
      </div>
    </div>
  );

  return (
    <div className="task-board">
      <div className="task-column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <TaskCard key={task._id} task={task} />
        ))}
      </div>
      <div className="task-column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <TaskCard key={task._id} task={task} />
        ))}
      </div>
      <div className="task-column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <TaskCard key={task._id} task={task} />
        ))}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalyticsDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutUsers: [],
    projectDelays: []
  });
  
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch users for risk analysis
      const usersResponse = await axios.get(`${API_URL}/api/users`);
      const users = usersResponse.data.data;
      
      // Analyze each user for risk
      const riskPromises = users.map(async (user) => {
        const activityData = {
          user_id: user._id,
          login_attempts: user.loginAttempts || 0,
          failed_logins: user.failedLogins || 0,
          unusual_hours: user.unusualHours || 0,
          data_access_volume: user.dataAccess || 0,
          permission_changes: user.permissionChanges || 0
        };
        
        const risk = await axios.post(
          `${ML_API_URL}/analyze-risk`,
          activityData
        );
        
        return { ...user, riskData: risk.data };
      });
      
      const riskResults = await Promise.all(riskPromises);
      const highRiskUsers = riskResults.filter(
        u => u.riskData.risk_level === 'high'
      );
      
      setAnalytics(prev => ({ ...prev, riskUsers: highRiskUsers }));
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI-Powered Analytics</h2>
      
      <div className="analytics-section">
        <h3>High Risk Users</h3>
        {analytics.riskUsers.map(user => (
          <div key={user._id} className="alert-card risk">
            <h4>{user.name}</h4>
            <p>Risk Score: {(user.riskData.score * 100).toFixed(1)}%</p>
            <ul>
              {user.riskData.factors.map((factor, idx) => (
                <li key={idx}>{factor}</li>
              ))}
            </ul>
          </div>
        ))}
      </div>
      
      <div className="analytics-section">
        <h3>Burnout Risk</h3>
        {analytics.burnoutUsers.map(user => (
          <div key={user._id} className="alert-card burnout">
            <h4>{user.name}</h4>
            <p>Burnout Risk: {user.burnoutData.burnout_risk}</p>
            <ul>
              {user.burnoutData.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Common Patterns

### Protected Routes

```javascript
// frontend/src/components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) {
    return <div>Loading...</div>;
  }

  if (!user) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Middleware for Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  try {
    let token;

    if (req.headers.authorization?.startsWith('Bearer')) {
      token = req.headers.authorization.split(' ')[1];
    }

    if (!token) {
      return res.status(401).json({ message: 'Not authorized' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');

    if (!req.user) {
      return res.status(401).json({ message: 'User not found' });
    }

    next();
  } catch (error) {
    res.status(401).json({ message: 'Not authorized' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: `User role ${req.user.role} is not authorized`
      });
    }
    next();
  };
};
```

### Model Training and Persistence

```python
# ml-service/train_models.py
import joblib
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import os

def train_ticket_classifier():
    # Sample training data
    tickets = [
        ("Server down", "technical"),
        ("Cannot access database", "technical"),
        ("Invoice not received", "billing"),
        ("Payment failed", "billing"),
        ("General inquiry", "general"),
        ("Unauthorized access detected", "security")
    ]
    
    texts = [t[0] for t in tickets]
    labels = [t[1] for t in tickets]
    
    vectorizer = TfidfVectorizer(max_features=1000)
    X = vectorizer.fit_transform(texts)
    
    classifier = MultinomialNB()
    classifier.fit(X, labels)
    
    # Save models
    model_path = os.getenv('MODEL_PATH', './models')
    os.makedirs(model_path, exist_ok=True)
    
    joblib.dump(vectorizer, f'{model_path}/ticket_vectorizer.pkl')
    joblib.dump(classifier, f'{model_path}/ticket_classifier.pkl')
    
    print("Models trained and saved successfully")

if __name__ == "__main__":
    train_ticket_classifier()
```

## Configuration

### Database Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
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

// Hash password before
