# fasting-pet-backend
{
  "name": "fasting-pet-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.2",
    "mongoose": "^7.6.1"
  }
}
ini
Copy
Edit
PORT=5000
MONGODB_URI=tu_uri_mongodb
JWT_SECRET=una_clave_supersecreta
js
Copy
Edit
require('dotenv').config();
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');

const authRoutes = require('./routes/auth');
const fastRoutes = require('./routes/fasts');

const app = express();
app.use(cors());
app.use(express.json());

app.use('/api/auth', authRoutes);
app.use('/api/fasts', fastRoutes);

mongoose.connect(process.env.MONGODB_URI)
  .then(() => {
    app.listen(process.env.PORT, () => {
      console.log(`Servidor corriendo en el puerto ${process.env.PORT}`);
    });
  })
  .catch(err => console.error('Error de conexión a MongoDB', err));
js
Copy
Edit
const mongoose = require('mongoose');

const FastSchema = new mongoose.Schema({
  startTime: Date,
  endTime: Date,
  completed: Boolean
});

const UserSchema = new mongoose.Schema({
  username: { type: String, unique: true },
  password: String,
  petStyle: { type: String, default: '🐶' },
  petAlive: { type: Boolean, default: true },
  fasts: [FastSchema]
});

module.exports = mongoose.model('User', UserSchema);
js
Copy
Edit
const jwt = require('jsonwebtoken');

module.exports = function (req, res, next) {
  const token = req.header('Authorization');
  if (!token) return res.status(401).json({ msg: 'Token requerido' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    res.status(401).json({ msg: 'Token inválido' });
  }
};
js
Copy
Edit
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

router.post('/register', async (req, res) => {
  const { username, password } = req.body;
  try {
    const exists = await User.findOne({ username });
    if (exists) return res.json({ msg: 'Usuario ya existe' });

    const hashed = await bcrypt.hash(password, 10);
    const user = new User({ username, password: hashed });
    await user.save();
    res.json({ msg: 'Usuario registrado correctamente' });
  } catch {
    res.status(500).json({ msg: 'Error del servidor' });
  }
});

router.post('/login', async (req, res) => {
  const { username, password } = req.body;
  try {
    const user = await User.findOne({ username });
    if (!user) return res.json({ msg: 'Credenciales inválidas' });

    const match = await bcrypt.compare(password, user.password);
    if (!match) return res.json({ msg: 'Credenciales inválidas' });

    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET);
    res.json({ token });
  } catch {
    res.status(500).json({ msg: 'Error del servidor' });
  }
});

module.exports = router;
js
Copy
Edit
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const User = require('../models/User');

router.get('/me', auth, async (req, res) => {
  const user = await User.findById(req.user.id);
  if (!user) return res.status(404).json({ msg: 'Usuario no encontrado' });
  res.json(user);
});

router.post('/start', auth, async (req, res) => {
  const { duration } = req.body; // duración en segundos
  const user = await User.findById(req.user.id);
  const startTime = new Date();
  const endTime = new Date(startTime.getTime() + duration * 1000);

  user.fasts.push({ startTime, endTime, completed: false });
  await user.save();

  res.json({ fast: { startTime, endTime } });
});

router.post('/complete', auth, async (req, res) => {
  const user = await User.findById(req.user.id);
  const last = user.fasts[user.fasts.length - 1];
  if (!last || last.completed) return res.json({ msg: 'No hay ayuno activo' });

  const now = new Date();
  const success = now >= last.endTime;

  last.completed = success;
  if (!success) user.petAlive = false;

  await user.save();

  res.json({
    petAlive: user.petAlive,
    petStyle: user.petStyle,
    reward: success ? 'Medalla de ayuno 🏅' : null
  });
});

router.post('/customize', auth, async (req, res) => {
  const { petStyle } = req.body;
  const user = await User.findById(req.user.id);
  user.petStyle = petStyle;
  await user.save();
  res.json({ msg: 'Mascota actualizada' });
});

module.exports = router;
