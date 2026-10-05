import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';

dotenv.config();
const app = express();
app.use(cors());
app.use(express.json());

const senhaStore = [];
const chamadas = [];

const prioridade = ['SP', 'SE', 'SG'];

function dentroDoExpediente() {
  const h = new Date().getHours();
  return h >= 7 && h < 17;
}

function gerarNumero(tipo) {
  const d = new Date();
  const yy = String(d.getFullYear()).slice(-2);
  const mm = String(d.getMonth() + 1).padStart(2, '0');
  const dd = String(d.getDate()).padStart(2, '0');
  const qtd = senhaStore.filter(s => s.tipo === tipo).length + 1;
  return `${yy}${mm}${dd}-${tipo}${String(qtd).padStart(3, '0')}`;
}

app.get('/api/health', (_req, res) => res.json({ status: 'ok' }));

app.post('/api/senhas', (req, res) => {
  const { tipo } = req.body;
  if (!['SP','SG','SE'].includes(tipo)) return res.status(400).json({ erro: 'Tipo inválido.' });
  const senha = {
    id: senhaStore.length + 1,
    numero: gerarNumero(tipo),
    tipo,
    estado: 'AGUARDANDO',
    emitidaEm: new Date().toISOString()
  };
  senhaStore.push(senha);
  res.status(201).json(senha);
});

function proximaSenha() {
  const ultima = chamadas.at(-1)?.tipo;
  const candidatas = senhaStore.filter(s => s.estado === 'AGUARDANDO');

  if (!candidatas.length) return null;

  const ordem = ultima === 'SP' ? ['SE','SG'] : ['SP','SE','SG'];
  for (const tipo of ordem) {
    const item = candidatas.find(s => s.tipo === tipo);
    if (item) return item;
  }
  return candidatas[0];
}

app.post('/api/atendimentos/chamar', (req, res) => {
  if (!dentroDoExpediente()) return res.status(400).json({ erro: 'Fora do expediente.' });
  const { guiche = 1, atendente = 'AA' } = req.body;
  const senha = proximaSenha();
  if (!senha) return res.status(404).json({ erro: 'Não há senhas aguardando.' });

  senha.estado = 'CHAMADA';
  senha.guiche = guiche;
  senha.atendente = atendente;
  senha.primeiraChamada = new Date().toISOString();
  chamadas.push(senha);
  res.json(senha);
});

app.post('/api/atendimentos/:id/iniciar', (req, res) => {
  const senha = senhaStore.find(s => s.id === Number(req.params.id));
  if (!senha || !['CHAMADA','CHAMADA_NOVAMENTE'].includes(senha.estado))
    return res.status(404).json({ erro: 'Senha não disponível para início.' });
  senha.estado = 'EM_ATENDIMENTO';
  senha.inicio = new Date().toISOString();
  res.json(senha);
});

app.post('/api/atendimentos/:id/finalizar', (req, res) => {
  const senha = senhaStore.find(s => s.id === Number(req.params.id));
  if (!senha || senha.estado !== 'EM_ATENDIMENTO')
    return res.status(404).json({ erro: 'Atendimento não encontrado.' });
  senha.estado = 'ATENDIDA';
  senha.finalizacao = new Date().toISOString();
  res.json(senha);
});

app.post('/api/atendimentos/:id/novamente', (req, res) => {
  const senha = senhaStore.find(s => s.id === Number(req.params.id));
  if (!senha || senha.estado !== 'CHAMADA')
    return res.status(404).json({ erro: 'Senha não pode ser chamada novamente.' });
  senha.estado = 'CHAMADA_NOVAMENTE';
  senha.segundaChamada = new Date().toISOString();
  res.json(senha);
});

app.get('/api/painel', (_req, res) => {
  res.json(chamadas.slice(-5).reverse());
});

app.listen(process.env.PORT || 3000, () => {
  console.log(`nassauTickets backend em http://localhost:${process.env.PORT || 3000}`);
});
