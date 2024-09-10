/
    ├── index.js
    ├── models.js
    │   ├── User.js
    │   ├── ThirdParty.js
    │   ├── Wallet.js
    │   ├── Transaction.js
    │   ├── Point.js
    │   └── Purchase.js
    ├── routes
    │   ├── users.js
    │   ├── thirdParties.js
    │   ├── points.js
    │   ├── purchases.js
    │   ├── usersBackOffice.js
    │   └── wallet.js
    └── config.js // Para almacenar variables de entorno (clave secreta de JWT, clave secreta de Stripe, etc.)
    ```

* **`index.js`:**
    ```javascript
    const express = require('express');
    const mongoose = require('mongoose');
    const cors = require('cors');
    const bcrypt = require('bcrypt');
    const jwt = require('jsonwebtoken');
    const userRouter = require('./routes/users');
    const thirdPartyRouter = require('./routes/thirdParties');
    const pointsRouter = require('./routes/points');
    const purchasesRouter = require('./routes/purchases');
    const userBackOfficeRouter = require('./routes/usersBackOffice');
    const walletRouter = require('./routes/wallet');

    const app = express();
    const PORT = process.env.PORT || 5000;

    // Middleware para permitir solicitudes desde otras direcciones (CORS)
    app.use(cors());
    app.use(express.json());

    // Conexión a la base de datos MongoDB (reemplaza con tu URL)
    mongoose.connect('mongodb://localhost:27017/tu_base_de_datos', {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    })
      .then(() => console.log('Conectado a la base de datos'))
      .catch(err => console.error('Error al conectar a la base de datos:', err));

    // Función para generar un token JWT
    const generateToken = (id) => {
      return jwt.sign({ id }, process.env.JWT_SECRET, { expiresIn: '1d' });
    };

    // Middleware de autenticación
    const authMiddleware = async (req, res, next) => {
      const token = req.header('Authorization');
      if (!token) return res.status(401).json({ message: 'Acceso no autorizado' });

      try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = await User.findById(decoded.id);
        next();
      } catch (error) {
        res.status(400).json({ message: 'Token inválido' });
      }
    };

    // Rutas para registrar usuarios
    app.post('/register', async (req, res) => {
      const { name, email, password } = req.body;

      try {
        const existingUser = await User.findOne({ email: email });
        if (existingUser) {
          return res.status(400).json({ message: 'El correo electrónico ya está en uso' });
        }
        const salt = await bcrypt.genSalt(10);
        const hashedPassword = await bcrypt.hash(password, salt);

        const newUser = new User({ name: name, email: email, password: hashedPassword });
        await newUser.save();

        const token = generateToken(newUser._id);

        res.json({ message: 'Usuario registrado correctamente', token: token });
      } catch (err) {
        res.status(500).json({ message: err.message });
      }
    });

    // Rutas para iniciar sesión
    app.post('/login', async (req, res) => {
      const { email, password } = req.body;

      try {
        const user = await User.findOne({ email: email });
        if (!user) {
          return res.status(400).json({ message: 'Usuario no encontrado' });
        }
        const isValidPassword = await bcrypt.compare(password, user.password);
        if (!isValidPassword) {
          return res.status(400).json({ message: 'Contraseña incorrecta' });
        }

        const token = generateToken(user._id);

        res.json({ message: 'Inicio de sesión correcto', token: token });
      } catch (err) {
        res.status(500).json({ message: err.message });
      }
    });

    // Rutas para registrar terceros
    app.post('/thirdParty/register', async (req, res) => {
      const { name, apiId, apiUrl } = req.body;

      try {
        const existingThirdParty = await ThirdParty.findOne({ apiId: apiId });
        if (existingThirdParty) {
          return res.status(400).json({ message: 'El ID de API ya está en uso' });
        }
        const newThirdParty = new ThirdParty({ name: name, apiId: apiId, apiUrl: apiUrl });
        await newThirdParty.save();

        const token = generateToken(newThirdParty._id);

        res.json({ message: 'Tercero registrado correctamente', token: token });
      } catch (err) {
        res.status(500).json({ message: err.message });
      }
    });

    // Rutas para iniciar sesión de terceros
    app.post('/thirdParty/login', async (req, res) => {
      const { apiId } = req.body;

      try {
        const thirdParty = await ThirdParty.findOne({ apiId: apiId });
        if (!thirdParty) {
          return res.status(400).json({ message: 'Tercero no encontrado' });
        }

        const token = generateToken(thirdParty._id);

        res.json({ message: 'Inicio de sesión correcto', token: token });
      } catch (err) {
        res.status(500).json({ message: err.message });
      }
    });

    // Rutas de la API protegidas con el middleware de autenticación
    app.use('/users', authMiddleware, userRouter);
    app.use('/thirdParties', authMiddleware, thirdPartyRouter); // Protección para terceros
    app.use('/points', authMiddleware, pointsRouter);
    app.use('/purchases', authMiddleware, purchasesRouter);
    app.use('/users/backoffice', authMiddleware, userBackOfficeRouter);
    app.use('/wallet', authMiddleware, walletRouter); // Protección para la billetera

    app.listen(PORT, () => console.log(`Servidor corriendo en el puerto ${PORT}`));
    ```

* **`models/User.js`:**
    ```javascript
    const mongoose = require('mongoose');

    const userSchema = new mongoose.Schema({
      name: { type: String, required: true },
      email: { type: String, required: true, unique: true },
      password: { type: String, required: true },
      sponsorId: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
      front: { type: Number, default: 0 },
      level: { type: Number, default: 1 },
      position: { type: Number, default: 0 },
      points: { type: Number, default: 0 },
      wallet: { type: mongoose.Schema.Types.ObjectId, ref: 'Wallet' },
      // ... otros campos del usuario
    });

    const User = mongoose.model('User', userSchema);

    module.exports = User;
    ```

* **`models/ThirdParty.js`:**
    ```javascript
    const mongoose = require('mongoose');

    const thirdPartySchema = new mongoose.Schema({
      name: { type: String, required: true },
      apiId: { type: String, required: true },
      apiUrl: { type: String, required: true },
      wallet: { type: mongoose.Schema.Types.ObjectId, ref: 'Wallet' },
      // ... otros campos del tercero
    });

    const ThirdParty = mongoose.model('ThirdParty', thirdPartySchema);

    module.exports = ThirdParty;
    ```

* **`models/Wallet.js`:**
    ```javascript
    const mongoose = require('mongoose');

    const walletSchema = new mongoose.Schema({
      userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
      thirdPartyId: { type: mongoose.Schema.Types.ObjectId, ref: 'ThirdParty' },
      balance: { type: Number, default: 0 },
      transactions: [
        {
          type: mongoose.Schema.Types.ObjectId,
          ref: 'Transaction'
        }
      ]
    });

    const Wallet = mongoose.model('Wallet', walletSchema);

    module.exports = Wallet;
    ```

* **`models/Transaction.js`:**
    ```javascript
    const mongoose = require('mongoose');

    const transactionSchema = new mongoose.Schema({
      walletId: { type: mongoose.Schema.Types.ObjectId, ref: 'Wallet', required: true },
      type: { type: String, enum: ['deposit', 'withdrawal', 'payment'], required: true },
      amount: { type: Number, required: true },
      date: { type: Date, default: Date.now },
      description: { type: String }
    });

    const Transaction = mongoose.model('Transaction', transactionSchema);

    module.exports = Transaction;
    ```

* **`models/Point.js`:**
    ```javascript
    const mongoose = require('mongoose');

    const pointSchema = new mongoose.Schema({
      userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
      thirdPartyId: { type: mongoose.Schema.Types.ObjectId, ref: 'ThirdParty' },
      points: { type: Number, required: true },
      date: { type: Date, default: Date.now },
      // ... otros campos del punto
    });

    const Point = mongoose.model('Point', pointSchema);

    module.exports = Point;
    ```

* **`models/Purchase.js`:**
    ```javascript
    const mongoose = require('mongoose');

    const purchaseSchema = new mongoose.Schema({
      userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
      thirdPartyId: { type: mongoose.Schema.Types.ObjectId, ref: 'ThirdParty', required: true },
      purchaseValue: { type: Number, required: true },
      date: { type: Date, default: Date.now },
      // ... otros campos de la compra
    });

    const Purchase = mongoose.model('Purchase', purchaseSchema);

    module.exports = Purchase;
    ```

* **`routes/users.js`:**
    ```javascript
    const express = require('express');
    const router = express.Router();
    const User = require('../models/User');
    const Wallet = require('../models/Wallet');

    // Crear un nuevo usuario
    router.post('/', async (req, res) => {
      const { name, email, password } = req.body;
      const sponsorId = req.body.sponsorId; // ID del patrocinador

      try {
        const existingUser = await User.findOne({ email: email });
        if (existingUser) {
          return res.status(400).json({ message: 'El correo electrónico ya está en uso' });
        }

        // Encontrar la posición correcta en la matriz
        const position = findPosition(sponsorId);

        // Crear la billetera para el nuevo usuario
        const newWallet = new Wallet({ userId: user._id });
        await newWallet.save();

        // Crear el nuevo usuario
        const newUser = new User({
          name: name,
          email: email,
          password: password,
          sponsorId: sponsorId,
          front: position.front,
          level: position.level,
          position: position.position,
          wallet: newWallet._id
        });
        await newUser.save();

        res.status(201).json(newUser);
      } catch (err) {
        res.status(400).json({ message: err.message });
      }
    });

    // Obtener todos los usuarios
    router.get('/', async (req, res) => {
      try {
        const users = await User.find().populate('wallet');
        res.json(users);
      } catch (err) {
        res.status(500).json({ message: err.message });
      }
    });

    // Obtener un usuario por ID
    router.get('/:id', getUser, (req, res) => {
      res.json(res.user);
    });

    // Actualizar un usuario por ID
    router.patch('/:id', getUser, async (req, res) => {
      try {
        const updatedUser = await User.updateOne(
          { _id: req.params.id },
          { $set: req.body }
        );
        res.json(updatedUser);
      } catch (err) {
        res.status(400).json({ message: err.message });
      }
    });

    // Eliminar un usuario por ID
    router.delete('/:id', getUser, async (req, res) => {
      try {
        await User.deleteOne({ _id: req.params.id });
        res.json({ message: 'Usuario eliminado' });
      } catch (err) {
        res.status(500).json({ message: err.message });
      }
    });

    // Función intermedia para obtener un usuario por ID
    async function getUser(req, res, next) {
      try {
        const user = await User.findById(req.params.id).populate('wallet');
        if (user == null) {
          return res.status(404).json({ message: 'Usuario no encontrado' });
        }
        res.user = user;
        next();
      } catch (err) {
        return res.status(500).json({ message: err.message });
      }
    }

    // Función para encontrar la posición en la matriz
    async function findPosition(sponsorId) {
      let position = { front: 0, level: 1, position: 0 }; // Posición inicial

      if (sponsorId) {
        const sponsor = await User.findById(sponsorId);

        // Si el patrocinador está en el nivel 10, el nuevo usuario va al nivel 11
        if (sponsor.level === 10) {
          position.level = 11; // (Puedes manejar este caso como quieras)
          return position;
        }

        // Buscar la posición libre en el siguiente nivel
        for (let front = 0; front < 3; front++) {
          for (let pos = 0; pos < 10; pos++) {
            const user = await User.findOne({
              front: front,
              level: sponsor.level + 1,
              position: pos,
            });

            if (!user) {
              position.front = front;
              position.level = sponsor.level + 1;
              position.position = pos;
              return position;
            }
          }
        }
      }

      return position; // Si no se encuentra una posición libre,  devolver la inicial
    }

    module.exports = router;
    ```

* **`routes/thirdParties.js`:**
    ```javascript
    const express = require('express');
    const router = express.Router();
    const ThirdParty = require('../models/ThirdParty');
    const Wallet = require('../models/Wallet');

    // Crear un nuevo tercero
    router.post('/', async (req, res) => {
      const { name, apiId, apiUrl } = req.body;

      try {
        const existingThirdParty = await ThirdParty.findOne({ apiId: apiId });
        if (existingThirdParty) {
          return res.status(400).json({ message: 'El ID de API ya está en uso' });
        }

        // Crear la billetera para el nuevo tercero
        const newWallet = new Wallet({ thirdPartyId: thirdParty._id });
        await newWallet.save();

        // Crear el nuevo tercero
        const newThirdParty = new ThirdParty({
          name: name,
          apiId: apiId,
          apiUrl: apiUrl,
          wallet: newWallet._id
        });
        await newThirdParty.save();

        res.status(201).json(newThirdParty);
      } catch (err) {
        res.status(400).json({ message: err.message });
      }
    });

    // Obtener todos los terceros
    router.get('/', async (req, res) => {
      try {
        const thirdParties = await ThirdParty.find().populate('wallet');
        res.json(thirdParties);
      } catch (err) {
        res.status(500).json({ message: err.message });
      }
    });

    // Obtener un tercero por ID
    router.get('/:id', getThirdParty, (req, res) => {
      res.json(res.thirdParty);
    });

    // Actualizar un tercero por ID
    router.patch('/:id', getThirdParty, async (req, res) => {
      try {
        const updatedThirdParty = await ThirdParty.updateOne(
          { _id: req.params.id },
          { $set: req.body }
        );
        res.json(updatedThirdParty);
      } catch (err) {
        res.status(400).json({ message: err.message });
[3:47 p. m., 8/9/2024] Venta de Pantallas Streaming: ¡Tienes razón!  Me he olvidado de incluir las rutas para procesar las compras y generar puntos.  

Aquí te dejo las rutas para procesar las compras y generar puntos en el archivo `routes/purchases.js`:

```javascript
// routes/purchases.js
const express = require('express');
const router = express.Router();
const Purchase = require('../models/Purchase');
const User = require('../models/User');
const Point = require('../models/Point');

// Crear una nueva compra
router.post('/', async (req, res) => {
  const userId = req.body.userId;
  const thirdPartyId = req.body.thirdPartyId;
  const purchaseValue = req.body.purchaseValue;

  try {
    const purchase = new Purchase({
      userId: userId,
      thirdPartyId: thirdPartyId,
      purchaseValue: purchaseValue
    });
    const savedPurchase = await purchase.save();

    // Calcular los puntos ganados por la compra
    const pointsEarned = calculatePoints(purchaseValue); 

    // Registrar los puntos ganados para el usuario
    await Point.create({ userId: userId, thirdPartyId: thirdPartyId, points: pointsEarned });

    // Actualizar el saldo de la billetera del usuario
    const user = await User.findById(userId).populate('wallet');
    user.wallet.balance += purchaseValue;
    await user.save();

    // Distribuir la comisión a la plataforma y al patrocinador
    const commission = calculateCommission(purchaseValue);
    const sponsor = await User.findById(user.sponsorId);
    const thirdParty = await ThirdParty.findById(thirdPartyId); // Obtener el tercero

    if (sponsor) {
      sponsor.wallet.balance += commission.sponsorCommission;
      await sponsor.save();
    }

    // Distribuir la comisión al tercero
    if (thirdParty) {
      thirdParty.wallet.balance += commission.thirdPartyCommission;
      await thirdParty.save();
    }

    res.status(201).json(savedPurchase);
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
});

// Función para calcular puntos ganados por la compra
async function calculatePoints(purchaseValue) {
  // Define aquí la lógica para calcular los puntos 
  // a partir del valor de la compra
  // Puedes usar una fórmula fija o una lógica más compleja
  // (por ejemplo,  un porcentaje del valor de la compra)

  // Ejemplo simple: 1 punto por cada $1 de compra
  const points = purchaseValue; 

  return points;
}

// Función para calcular la comisión
async function calculateCommission(purchaseValue) {
  // Implementar la lógica para calcular las comisiones
  // Ejemplo: 5% para la plataforma, 2% para el usuario, 1% para el patrocinador
  const commission = {
    platformCommission: purchaseValue * 0.05,
    sponsorCommission: purchaseValue * 0.01,
    thirdPartyCommission: purchaseValue * 0.02 // Ejemplo de comisión para el tercero
  };

  return commission;
}

module.exports = router;
```

También debes agregar estas rutas al archivo `index.js` del backend:

```javascript
// index.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const userRouter = require('./routes/users');
const thirdPartyRouter = require('./routes/thirdParties');
const pointsRouter = require('./routes/points');
const purchasesRouter = require('./routes/purchases');
const userBackOfficeRouter = require('./routes/usersBackOffice');
const walletRouter = require('./routes/wallet');

// ... (resto del código)

// Rutas de la API
app.use('/users', authMiddleware, userRouter);
app.use('/thirdParties', authMiddleware, thirdPartyRouter); // Protección para terceros
app.use('/points', authMiddleware, pointsRouter);
app.use('/purchases', authMiddleware, purchasesRouter); // Agrega esta ruta
app.use('/users/backoffice', authMiddleware, userBackOfficeRouter);
app.use('/wallet', authMiddleware, walletRouter); // Protección para la billetera

// ... (resto del código)```
