const express = require('express');
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const UserSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  auctions: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Auction' }]
});

UserSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

UserSchema.methods.comparePassword = function(password) {
  return bcrypt.compare(password, this.password);
};

UserSchema.methods.generateAuthToken = function() {
  return jwt.sign({ _id: this._id }, 'secret_key');
};

const User = mongoose.model('User', UserSchema);

const app = express();
app.use(express.json());

app.post('/register', async (req, res) => {
  const user = new User(req.body);
  await user.save();
  const token = user.generateAuthToken();
  res.header('x-auth-token', token).send({
    _id: user._id,
    email: user.email
  });
});

app.post('/login', async (req, res) => {
  const user = await User.findOne({ email: req.body.email });
  if (!user) return res.status(400).send('Invalid email or password.');

  const validPassword = await user.comparePassword(req.body.password);
  if (!validPassword) return res.status(400).send('Invalid email or password.');

  const token = user.generateAuthToken();
  res.send(token);
});

app.listen(3000, () => console.log('Server started on port 3000'));
