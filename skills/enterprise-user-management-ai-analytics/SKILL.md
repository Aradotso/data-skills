---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly analysis, and predictive insights
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics with user management
  - implement JWT authentication for user dashboard
  - create admin dashboard with role-based access
  - build kanban board for task tracking
  - add AI-powered ticket classification
  - detect user anomalies and burnout
  - setup ML service for predictive analytics
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack JavaScript application that combines user/task management with AI-powered analytics. It provides:

- **User Management**: Secure authentication, role-based access, task tracking
- **Admin Features**: User CRUD operations, task assignment, ticket management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, predictive insights
- **Task Management**: Kanban boards, time tracking, progress monitoring

The system uses React.js frontend, Node.js backend, MongoDB database, and FastAPI ML service with scikit-learn and River for online learning.

## Installation

### Prerequisites

- Node.js 14+
- Python 3.8+
- MongoDB
- npm or yarn

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
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
# Frontend runs at http://localhost:3000
```

## Architecture

```
├── frontend/          # React.js application
│   ├── src/
│   │   ├── components/  # Reusable UI components
│   │   ├── pages/       # Dashboard, Kanban, Tickets
│   │   ├── services/    # API calls
│   │   └── utils/       # Helpers, auth
├── backend/           # Node.js REST API
│   ├── models/        # MongoDB schemas
│   ├── routes/        # API endpoints
│   ├── middleware/    # Auth, validation
│   └── controllers/   # Business logic
└── ml-service/        # FastAPI ML service
    ├── models/        # Trained models
    ├── services/      # AI logic
    └── main.py        # FastAPI app
```

## Core Features & Code Examples

### 1. User Authentication (JWT)

**Backend - User Login Controller:**

```javascript
// backend/controllers/authController.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Find user and validate password
    const user = await User.findOne({ email }).select('+password');
    if (!user || !(await user.matchPassword(password))) {
      return res.status(401).json({ 
        success: false, 
        message: 'Invalid credentials' 
      });
    }
    
    // Generate JWT token
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(200).json({
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
    res.status(500).json({ success: false, message: error.message });
  }
};
```

**Frontend - Auth Service:**

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
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  }
  
  logout() {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  }
  
  getCurrentUser() {
    return JSON.parse(localStorage.getItem('user'));
  }
  
  getToken() {
    return localStorage.getItem('token');
  }
}

export default new AuthService();
```

**Frontend - Protected Route:**

```javascript
// frontend/src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import authService from '../services/authService';

const ProtectedRoute = ({ children, requiredRole }) => {
  const user = authService.getCurrentUser();
  const token = authService.getToken();
  
  if (!token || !user) {
    return <Navigate to="/login" replace />;
  }
  
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" replace />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### 2. User Management (Admin)

**Backend - User CRUD:**

```javascript
// backend/controllers/userController.js
const User = require('../models/User');

// Get all users (Admin only)
exports.getUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    
    res.status(200).json({
      success: true,
      count: users.length,
      data: users
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Create user
exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const user = await User.create({
      name,
      email,
      password,
      role: role || 'user',
      department
    });
    
    res.status(201).json({
      success: true,
      data: user
    });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ 
        success: false, 
        message: 'User not found' 
      });
    }
    
    res.status(200).json({
      success: true,
      data: user
    });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};

// Delete user
exports.deleteUser = async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    
    if (!user) {
      return res.status(404).json({ 
        success: false, 
        message: 'User not found' 
      });
    }
    
    res.status(200).json({
      success: true,
      message: 'User deleted successfully'
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

**Frontend - User Management Component:**

```javascript
// frontend/src/pages/UserManagement.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import authService from '../services/authService';

const UserManagement = () => {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [showModal, setShowModal] = useState(false);
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    password: '',
    role: 'user',
    department: ''
  });

  const API_URL = process.env.REACT_APP_API_URL;
  const token = authService.getToken();

  const fetchUsers = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/users`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setUsers(response.data.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching users:', error);
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchUsers();
  }, []);

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await axios.post(`${API_URL}/api/users`, formData, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setShowModal(false);
      fetchUsers();
      setFormData({ name: '', email: '', password: '', role: 'user', department: '' });
    } catch (error) {
      console.error('Error creating user:', error);
    }
  };

  const handleDelete = async (userId) => {
    if (window.confirm('Are you sure you want to delete this user?')) {
      try {
        await axios.delete(`${API_URL}/api/users/${userId}`, {
          headers: { Authorization: `Bearer ${token}` }
        });
        fetchUsers();
      } catch (error) {
        console.error('Error deleting user:', error);
      }
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="user-management">
      <div className="header">
        <h2>User Management</h2>
        <button onClick={() => setShowModal(true)}>Add User</button>
      </div>

      <table className="user-table">
        <thead>
          <tr>
            <th>Name</th>
            <th>Email</th>
            <th>Role</th>
            <th>Department</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {users.map(user => (
            <tr key={user._id}>
              <td>{user.name}</td>
              <td>{user.email}</td>
              <td>{user.role}</td>
              <td>{user.department}</td>
              <td>
                <button onClick={() => handleDelete(user._id)}>Delete</button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>

      {showModal && (
        <div className="modal">
          <form onSubmit={handleSubmit}>
            <h3>Add New User</h3>
            <input
              type="text"
              placeholder="Name"
              value={formData.name}
              onChange={(e) => setFormData({...formData, name: e.target.value})}
              required
            />
            <input
              type="email"
              placeholder="Email"
              value={formData.email}
              onChange={(e) => setFormData({...formData, email: e.target.value})}
              required
            />
            <input
              type="password"
              placeholder="Password"
              value={formData.password}
              onChange={(e) => setFormData({...formData, password: e.target.value})}
              required
            />
            <select
              value={formData.role}
              onChange={(e) => setFormData({...formData, role: e.target.value})}
            >
              <option value="user">User</option>
              <option value="admin">Admin</option>
            </select>
            <input
              type="text"
              placeholder="Department"
              value={formData.department}
              onChange={(e) => setFormData({...formData, department: e.target.value})}
            />
            <div className="actions">
              <button type="submit">Create</button>
              <button type="button" onClick={() => setShowModal(false)}>Cancel</button>
            </div>
          </form>
        </div>
      )}
    </div>
  );
};

export default UserManagement;
```

### 3. Task Management with Kanban Board

**Backend - Task Model & Controller:**

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  status: {
    type: String,
    enum: ['todo', 'inProgress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
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
  timeSpent: {
    type: Number,
    default: 0
  },
  tags: [String]
}, {
  timestamps: true
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Get tasks by user
exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name');
    
    res.status(200).json({
      success: true,
      data: tasks
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Create task
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
    res.status(400).json({ success: false, message: error.message });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ 
        success: false, 
        message: 'Task not found' 
      });
    }
    
    res.status(200).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};

// Update time spent
exports.updateTimeSpent = async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent } },
      { new: true }
    );
    
    res.status(200).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};
```

**Frontend - Kanban Board:**

```javascript
// frontend/src/pages/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import authService from '../services/authService';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [activeTimer, setActiveTimer] = useState(null);
  const [timerSeconds, setTimerSeconds] = useState(0);

  const API_URL = process.env.REACT_APP_API_URL;
  const token = authService.getToken();
  const user = authService.getCurrentUser();

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks/user/${user.id}`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      const grouped = {
        todo: response.data.data.filter(t => t.status === 'todo'),
        inProgress: response.data.data.filter(t => t.status === 'inProgress'),
        done: response.data.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  useEffect(() => {
    fetchTasks();
  }, []);

  useEffect(() => {
    let interval;
    if (activeTimer) {
      interval = setInterval(() => {
        setTimerSeconds(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [activeTimer]);

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    try {
      await axios.patch(`${API_URL}/api/tasks/${taskId}/status`, 
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` }}
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    setTimerSeconds(0);
  };

  const stopTimer = async (taskId) => {
    try {
      await axios.patch(`${API_URL}/api/tasks/${taskId}/time`,
        { timeSpent: timerSeconds },
        { headers: { Authorization: `Bearer ${token}` }}
      );
      setActiveTimer(null);
      setTimerSeconds(0);
      fetchTasks();
    } catch (error) {
      console.error('Error updating time:', error);
    }
  };

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const renderColumn = (status, title) => (
    <div 
      className="kanban-column"
      onDrop={(e) => handleDrop(e, status)}
      onDragOver={handleDragOver}
    >
      <h3>{title} ({tasks[status].length})</h3>
      <div className="task-list">
        {tasks[status].map(task => (
          <div 
            key={task._id}
            className="task-card"
            draggable
            onDragStart={(e) => handleDragStart(e, task._id)}
          >
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <div className="task-meta">
              <span className={`priority ${task.priority}`}>{task.priority}</span>
              <span className="time">⏱ {formatTime(task.timeSpent)}</span>
            </div>
            {status === 'inProgress' && (
              <div className="timer">
                {activeTimer === task._id ? (
                  <>
                    <span>{formatTime(timerSeconds)}</span>
                    <button onClick={() => stopTimer(task._id)}>Stop</button>
                  </>
                ) : (
                  <button onClick={() => startTimer(task._id)}>Start Timer</button>
                )}
              </div>
            )}
          </div>
        ))}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <h2>My Tasks</h2>
      <div className="kanban-columns">
        {renderColumn('todo', 'To Do')}
        {renderColumn('inProgress', 'In Progress')}
        {renderColumn('done', 'Done')}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### 4. AI-Powered Ticket Classification

**ML Service - Ticket Classifier:**

```python
# ml-service/services/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

class TicketClassifier:
    def __init__(self):
        self.model_path = os.getenv('MODEL_PATH', './models')
        self.vectorizer = TfidfVectorizer(max_features=1000)
        self.classifier = MultinomialNB()
        self.categories = ['technical', 'billing', 'account', 'general']
        self.load_or_train()
    
    def load_or_train(self):
        try:
            self.vectorizer = joblib.load(f'{self.model_path}/vectorizer.pkl')
            self.classifier = joblib.load(f'{self.model_path}/classifier.pkl')
        except:
            # Initial training with sample data
            sample_data = [
                "Cannot login to my account", 
                "Payment failed",
                "How to reset password",
                "System is slow"
            ]
            sample_labels = ['account', 'billing', 'account', 'technical']
            self.train(sample_data, sample_labels)
    
    def train(self, texts, labels):
        X = self.vectorizer.fit_transform(texts)
        self.classifier.fit(X, labels)
        
        # Save models
        os.makedirs(self.model_path, exist_ok=True)
        joblib.dump(self.vectorizer, f'{self.model_path}/vectorizer.pkl')
        joblib.dump(self.classifier, f'{self.model_path}/classifier.pkl')
    
    def classify(self, text):
        X = self.vectorizer.transform([text])
        prediction = self.classifier.predict(X)[0]
        probabilities = self.classifier.predict_proba(X)[0]
        
        return {
            'category': prediction,
            'confidence': float(max(probabilities)),
            'probabilities': {
                cat: float(prob) 
                for cat, prob in zip(self.categories, probabilities)
            }
        }
    
    def update_model(self, text, actual_label):
        """Online learning - update model with new data"""
        X = self.vectorizer.transform([text])
        self.classifier.partial_fit(X, [actual_label])
        
        # Save updated model
        joblib.dump(self.classifier, f'{self.model_path}/classifier.pkl')
```

**ML Service - FastAPI Endpoints:**

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from services.ticket_classifier import TicketClassifier
from services.anomaly_detector import AnomalyDetector
from services.risk_predictor import RiskPredictor
import uvicorn

app = FastAPI(title="Enterprise ML Service")

# Initialize services
ticket_classifier = TicketClassifier()
anomaly_detector = AnomalyDetector()
risk_predictor = RiskPredictor()

class TicketRequest(BaseModel):
    text: str

class TicketFeedback(BaseModel):
    text: str
    actual_label: str

class UserActivityRequest(BaseModel):
    user_id: str
    login_time: str
    login_location: str
    device: str

class WorkloadRequest(BaseModel):
    user_id: str
    tasks_count: int
    avg_task_duration: float
    overtime_hours: float

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    try:
        result = ticket_classifier.classify(request.text)
        return {
            "success": True,
            "data": result
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/ticket-feedback")
async def ticket_feedback(request: TicketFeedback):
    try:
        ticket_classifier.update_model(request.text, request.actual_label)
        return {
            "success": True,
            "message": "Model updated successfully"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: UserActivityRequest):
    try:
        result = anomaly_detector.detect({
            'user_id': request.user_id,
            'login_time': request.login_time,
            'login_location': request.login_location,
            'device': request.device
        })
        return {
            "success": True,
            "data": result
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-burnout")
async def predict_burnout(request: WorkloadRequest):
    try:
        result = risk_predictor.predict_burnout({
            'user_id': request.user_id,
            'tasks_count': request.tasks_count,
            'avg_task_duration': request.avg_task_duration,
            'overtime_hours': request.overtime_hours
        })
        return {
            "success": True,
            "data": result
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**ML Service - Anomaly Detection:**

```python
# ml-service/services/anomaly_detector.py
from sklearn.ensemble import IsolationForest
from datetime import datetime
import joblib
import numpy as np
import os

class AnomalyDetector:
    def __init__(self):
        self.model_path = os.getenv('MODEL_PATH', './models')
        self.model = IsolationForest(contamination=0.1, random_state=42)
        self.user_profiles = {}
        self.load_model()
    
    def load_model(self):
        try:
            self.model = joblib.load(f'{self.model_path}/anomaly_detector.pkl')
            self.user_profiles = joblib.load(f'{self.model_path}/user_profiles.pkl')
        except:
            pass
    
    def extract_features(self, activity):
        """Convert activity data to numerical features"""
        # Parse login time to hour of day
        try:
            login_hour = datetime.fromisoformat(activity['login_time']).hour
        except:
            login_hour = 12  # default
        
        # Encode location (simplified - use hash)
        location_hash = hash(activity['login_location']) % 100
        
        # Encode device
        device_hash = hash(activity['device']) % 10
        
        return np.array([login_hour, location_hash, device_hash])
    
    def detect(self, activity):
        """Detect if activity is anomalous"""
        features = self.extract_features(activity)
        user_id = activity['user_id']
        
        # Get user's normal behavior
        if user_id not in self.user_profiles:
            self.user_profiles[user_id] = []
        
        # Check anomaly
        if len(self.user_profiles[user_id]) > 10:
            X = np.array(self.user_profiles[user_id])
            self.model.fit(X)
            prediction = self.model.predict([features])[0]
            score = self.model.score_samples([features])[0]
            
            is_anomaly = prediction == -1
            risk_level = 'high' if is_anomaly else 'low'
        else:
            is_anomaly = False
            risk_level = 'low'
            score = 0.0
        
        # Update user profile
        self.user_profiles[user_id].append(features.tolist())
        if len(self.user_profiles[user_id]) > 100:
            self.user_profiles[user_id].pop(0)
        
        # Save updated profiles
        os.makedirs(self.model_path, exist_ok=True)
        joblib.dump(self.user_profiles, f'{self.model_path}/user_profiles.pkl')
        
        return {
            'is_anomaly': bool(is_anomaly),
            'risk_level': risk_level,
            'anomaly_score': float(score),
            'reasons': self._get_anomaly_reasons(activity, is_anomaly)
        }
    
    def _get_anomaly_reasons(self, activity, is_anomaly):
        if not is_anomaly:
            return []
        
        reasons = []
        login_hour = datetime.fromisoformat(activity['login_time']).hour
        
        if login_hour < 6 or login_hour > 22:
            reasons.append('Unusual login time')
        
        # Add more heuristic checks
        return reasons
```

**ML Service -
