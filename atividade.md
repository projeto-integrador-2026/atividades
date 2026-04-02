# Atividade Avaliativa


# Guia Passo a Passo: Criando o CRUD de Usuários

Este guia conduzirá você para criação da gestão de usuários (CRUD) seguindo o padrão de arquitetura **MVC** utilizado em nosso projeto.

---

## 1. Banco de Dados

Antes de tudo, certifique-se de que a tabela `usuarios` existe no seu banco de dados. Caso não exista, execute o script abaixo:

```sql
CREATE TABLE IF NOT EXISTS usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome_usuario VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    senha VARCHAR(255) NOT NULL,
    perfil VARCHAR(20) NOT NULL DEFAULT 'user',
    criado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO usuarios (nome_usuario, email, senha, perfil)
VALUES ('admin', 'admin@email.com', '$2y$10$TKh8H1.PfQx37YgCzwiKb.KjNyWgaHb9cbcoQgdIVFlYg7B77UdFm', 'admin');

-- admin@email.com 
-- admin123
```

---

## 2. Criando um modelo de apresentação do Usuário no sistema. O Model Usuário (`app/models/Usuario.php`)

A Model representa a entidade "Usuário" do mundo real no nosso sistema.

```php
<?php

namespace app\models;

use DateTimeImmutable;

class Usuario
{
    private int $id;
    private string $nomeUsuario;
    private string $email;
    private string $senha;
    private string $perfil;
    private DateTimeImmutable $criadoEm;

    public function __construct(int $id = 0, string $nomeUsuario = '', string $email = '', string $senha = '', string $perfil = 'user')
    {
        $this->id = $id;
        $this->nomeUsuario = $nomeUsuario;
        $this->email = $email;
        $this->senha = $senha;
        $this->perfil = $perfil;
        $this->criadoEm = new DateTimeImmutable();
    }


    // Getters e Setters (omitidos para brevidade neste guia)
    public function getId(): int { return $this->id; }
    public function getNomeUsuario(): string { return $this->nomeUsuario; }
    public function getEmail(): string { return $this->email; }
    public function getSenha(): string { return $this->senha; }
    public function getPerfil(): string { return $this->perfil; }
    public function getCriadoEm(): DateTimeImmutable { return $this->criadoEm; }
}
```

---

## 3. O Repositório (`app/repositories/UsuarioRepository.php`)

O Repositório é responsável por conversar diretamente com o banco de dados via SQL.

```php
<?php

namespace app\repositories;

use app\database\ConnectionFactory;
use app\models\Usuario;
use PDO;

class UsuarioRepository
{
    private PDO $connection;

    public function __construct()
    {
        $this->connection = ConnectionFactory::getConnection();
    }

    public function getUsuarios(): array
    {
        $sql = "SELECT id, nome_usuario as nomeUsuario, email, perfil FROM usuarios";
        $stmt = $this->connection->prepare($sql);
        $stmt->execute();
        return $stmt->fetchAll();
    }

    public function saveUsuario(Usuario $usuario): bool
    {
        $sql = "INSERT INTO usuarios (nome_usuario, email, senha, perfil) VALUES (:nome, :email, :senha, :perfil)";
        $stmt = $this->connection->prepare($sql);
        $stmt->bindValue(':nome', $usuario->getNomeUsuario());
        $stmt->bindValue(':email', $usuario->getEmail());
        $stmt->bindValue(':senha', password_hash($usuario->getSenha(), PASSWORD_DEFAULT));
        $stmt->bindValue(':perfil', $usuario->getPerfil());
        return $stmt->execute();
    }
}
```

---

## 4. O Serviço (`app/services/UsuarioService.php`)

O Serviço contém as regras de negócio. Ele serve como uma ponte entre o Controller e o Repositório.

```php
<?php

namespace app\services;

use app\models\Usuario;
use app\repositories\UsuarioRepository;

class UsuarioService
{
    private UsuarioRepository $repository;

    public function __construct()
    {
        $this->repository = new UsuarioRepository();
    }

    public function getUsuarios(): array
    {
        return $this->repository->getUsuarios();
    }

    public function saveUsuario(Usuario $usuario): bool
    {
        return $this->repository->saveUsuario($usuario);
    }
}
```

---

## 5. O Controller (`app/controllers/UsuarioController.php`)

O Controller processa as requisições (URLs) e decide qual View mostrar ou qual Serviço chamar.

```php
<?php

namespace app\controllers;

use app\core\Controller;
use app\models\Usuario;
use app\services\UsuarioService;

class UsuarioController extends Controller
{
    private UsuarioService $service;

    public function __construct() {
        $this->service = new UsuarioService();
    }

    public function index() {
        $data['usuarios'] = $this->service->getUsuarios();
        $this->view('usuario/usuario_list', $data);
    }

    public function cadastrar() {
        $this->view('usuario/usuario_create');
    }

    public function salvar() {
        $usuario = new Usuario(
            0, 
            $_POST['nomeUsuario'], 
            $_POST['email'], 
            $_POST['senha'], 
            $_POST['perfil']
        );
        $this->service->saveUsuario($usuario);
        $this->redirect(URL_BASE . '/usuarios');
    }
}
```

---

## 6. A View de Listagem (`app/views/usuario/usuario_list.php`)

Use o Bootstrap 5 para criar uma tabela bonita e funcional.

```html
<div class="container mt-5">
    <h2>Lista de Usuários</h2>
    <a href="<?= URL_BASE ?>/usuarios/cadastrar" class="btn btn-primary mb-3">Novo Usuário</a>
    <table class="table table-striped">
        <thead>
            <tr>
                <th>Nome</th>
                <th>E-mail</th>
                <th>Perfil</th>
            </tr>
        </thead>
        <tbody>
            <?php foreach ($usuarios as $u): ?>
                <tr>
                    <td><?= $u['nomeUsuario'] ?></td>
                    <td><?= $u['email'] ?></td>
                    <td><?= $u['perfil'] ?></td>
                </tr>
            <?php endforeach; ?>
        </tbody>
    </table>
</div>
```

---

## 7. Rotas (`public/index.php`)

Por fim, vincule as URLs aos métodos do seu Controller.

O fluxo começa por aqui! Uma rota chama uma controller, que chama um método da controller, que chama um método do service, que chama um método do repositório, que chama um método do banco de dados.

```php
// Adicione em public/index.php
$router->get('/usuarios', 'UsuarioController@index');
$router->get('/usuarios/cadastrar', 'UsuarioController@cadastrar');
$router->post('/usuarios/salvar', 'UsuarioController@salvar');
```

---

## 8. Teste sua Aplicação

Teste sua aplicação, faça o cadastro de um usuário e verifique se ele foi salvo no banco de dados.

## 9. A View de Cadastro (`app/views/usuario/usuario_create.php`)

O formulário de cadastro coleta as informações do novo usuário para serem processadas pelo Controller.

```html
<div class="container mt-5">
    <div class="card shadow-sm col-md-6 mx-auto border-primary">
        <div class="card-header bg-primary text-white">
            <h4 class="mb-0">Novo Usuário</h4>
        </div>
        <div class="card-body">
            <form action="<?= URL_BASE ?>/usuarios/salvar" method="post">
                
                <div class="mb-3">
                    <label for="nomeUsuario" class="form-label">Nome de Usuário</label>
                    <input type="text" class="form-control" id="nomeUsuario" name="nomeUsuario" placeholder="Digite o nome de usuário">
                </div>

                <div class="mb-3">
                    <label for="email" class="form-label">E-mail</label>
                    <input type="email" class="form-control" id="email" name="email" placeholder="email@exemplo.com">
                </div>

                <div class="mb-3">
                    <label for="senha" class="form-label">Senha</label>
                    <input type="password" class="form-control" id="senha" name="senha" placeholder="Crie uma senha forte">
                </div>

                <div class="mb-3">
                    <label for="perfil" class="form-label">Perfil de Acesso</label>
                    <select class="form-select" id="perfil" name="perfil">
                        <option value="user" selected>Usuário Padrão</option>
                        <option value="admin">Administrador</option>
                    </select>
                </div>

                <div class="d-grid gap-2">
                    <button type="submit" class="btn btn-success">
                        <i class="bi bi-save"></i> Salvar Usuário
                    </button>
                    <a href="<?= URL_BASE ?>/usuarios" class="btn btn-outline-secondary">Cancelar</a>
                </div>
            </form>
        </div>
    </div>
</div>
```

---

## 10. Atualizando o Repositório e Serviço para Edição de Usuários

Para editar, precisamos primeiro **buscar** o usuário pelo ID e, depois, ter um método para **atualizar** seus dados no banco. Para tanto, vamos adicionar os seguintes métodos ao `UsuarioRepository.php`:

### app/repositories/UsuarioRepository.php

```php
public function getUsuarioById(int $id) {
    $sql = "SELECT id, nome_usuario as nomeUsuario, email, perfil FROM usuarios WHERE id = :id";
    $stmt = $this->connection->prepare($sql);
    $stmt->bindValue(':id', $id);
    $stmt->execute();
    return $stmt->fetch();
}

public function updateUsuario(Usuario $usuario): bool {
    $sql = "UPDATE usuarios SET nome_usuario = :nome, email = :email, perfil = :perfil WHERE id = :id";
    $stmt = $this->connection->prepare($sql);
    $stmt->bindValue(':nome', $usuario->getNomeUsuario());
    $stmt->bindValue(':email', $usuario->getEmail());
    $stmt->bindValue(':perfil', $usuario->getPerfil());
    $stmt->bindValue(':id', $usuario->getId());
    return $stmt->execute();
}
```

Vamos adicionar os seguintes métodos ao `UsuarioService.php`:

### app/services/UsuarioService.php

```php
public function getUsuarioById(int $id) {
    return $this->repository->getUsuarioById($id);
}

public function updateUsuario(Usuario $usuario): bool {
    return $this->repository->updateUsuario($usuario);
}
```

---

## 11. Novos Métodos no Controller (`app/controllers/UsuarioController.php`)

Adicione estes métodos para gerenciar a exibição do formulário (com dados preenchidos) e o processamento da atualização.

Aqui são necessário dois métodos: o método ```editar()``` que será responsável por exibir o formulário de edição e o método ```atualizar()``` que será responsável por processar a atualização dos dados do usuário.

```php
public function editar() {
    $id = $_GET['id'];
    $data['usuario'] = $this->service->getUsuarioById($id);
    $this->view('usuario/usuario_edit', $data);
}

public function atualizar() {
    $usuario = new Usuario(
        $_POST['id'], 
        $_POST['nomeUsuario'], 
        $_POST['email'], 
        $_POST['senha'] ?? '', 
        $_POST['perfil']
    );
    $this->service->updateUsuario($usuario);
    $this->redirect(URL_BASE . '/usuarios');
}
```

---

## 12. Novas Rotas (`public/index.php`)

Não esqueça de registrar as novas rotas de edição no seu arquivo de rotas.

```php
$router->get('/usuarios/editar', 'UsuarioController@editar');
$router->post('/usuarios/atualizar', 'UsuarioController@atualizar');
```

---

## 13. A View de Edição (`app/views/usuario/usuario_edit.php`)

Diferente do cadastro, a view de edição já vem com os campos preenchidos e aponta para a rota de atualização.

```html
<div class="container mt-5">
    <div class="card shadow-sm col-md-6 mx-auto border-primary">
        <div class="card-header bg-primary text-white">
            <h4 class="mb-0">Editar Usuário</h4>
        </div>
        <div class="card-body">
            <form action="<?= URL_BASE ?>/usuarios/atualizar" method="post">
                <!-- Campo oculto para o ID -->
                <input type="hidden" name="id" value="<?= $usuario['id'] ?>">

                <div class="mb-3">
                    <label for="nomeUsuario" class="form-label">Nome de Usuário</label>
                    <input type="text" class="form-control" id="nomeUsuario" name="nomeUsuario" value="<?= $usuario['nomeUsuario'] ?>">
                </div>

                <div class="mb-3">
                    <label for="email" class="form-label">E-mail</label>
                    <input type="email" class="form-control" id="email" name="email" value="<?= $usuario['email'] ?>">
                </div>

                <div class="mb-3">
                    <label for="senha" class="form-label">Nova Senha (Deixe em branco para não alterar)</label>
                    <input type="password" class="form-control" id="senha" name="senha">
                </div>

                <div class="mb-3">
                    <label for="perfil" class="form-label">Perfil de Acesso</label>
                    <select class="form-select" id="perfil" name="perfil">
                        <option value="user" <?= $usuario['perfil'] === 'user' ? 'selected' : '' ?>>Usuário Padrão</option>
                        <option value="admin" <?= $usuario['perfil'] === 'admin' ? 'selected' : '' ?>>Administrador</option>
                    </select>
                </div>

                <div class="d-grid gap-2">
                    <button type="submit" class="btn btn-primary">Atualizar Dados</button>
                    <a href="<?= URL_BASE ?>/usuarios" class="btn btn-outline-secondary">Cancelar</a>
                </div>
            </form>
        </div>
    </div>
</div>
```

---

## 14. Funcionalidade de Exclusão

Para fechar o nosso CRUD, vamos adicionar a funcionalidade de remover um usuário do sistema.

### Repository e Service

```php
// No UsuarioRepository.php
public function deleteUsuario(int $id): bool {
    $sql = "DELETE FROM usuarios WHERE id = :id";
    $stmt = $this->connection->prepare($sql);
    $stmt->bindValue(':id', $id);
    return $stmt->execute();
}

// No UsuarioService.php
public function deleteUsuario(int $id): bool {
    return $this->repository->deleteUsuario($id);
}
```

### Controller e Rotas

```php
// No UsuarioController.php
public function excluir() {
    $id = $_GET['id'];
    $this->service->deleteUsuario($id);
    $this->redirect(URL_BASE . '/usuarios');
}

// Em public/index.php
$router->get('/usuarios/excluir', 'UsuarioController@excluir');
```

## 15. Atualizando a View de Listagem (`app/views/usuario/usuario_list.php`)

Atualize a view de listagem para incluir os botões de **editar** e **excluir**.

```html
<div class="container mt-5">
    <h2>Lista de Usuários</h2>
    <a href="<?= URL_BASE ?>/usuarios/cadastrar" class="btn btn-primary mb-3">Novo Usuário</a>
    <table class="table table-striped">
        <thead>
            <tr>
                <th>Nome</th>
                <th>E-mail</th>
                <th>Perfil</th>
                <th>Ações</th>
            </tr>
        </thead>
        <tbody>
            <?php foreach ($usuarios as $u): ?>
                <tr>
                    <td><?= $u['nomeUsuario'] ?></td>
                    <td><?= $u['email'] ?></td>
                    <td><?= $u['perfil'] ?></td>
                    <td>
                        <a href="<?= URL_BASE ?>/usuarios/editar?id=<?= $u['id'] ?>" class="btn btn-primary btn-sm">
                            <i class="bi bi-pencil"></i>
                        </a>
                        <a href="<?= URL_BASE ?>/usuarios/excluir?id=<?= $u['id'] ?>" class="btn btn-danger btn-sm">
                            <i class="bi bi-trash"></i>
                        </a>
                    </td>
                </tr>
            <?php endforeach; ?>
        </tbody>
    </table>
</div>
```

# 16. Regras de negócio para salvar um usuário

Uma regra de negócio identificada em nosso sistema é que não pode existir dois usuários com o mesmo e-mail. Para salvar um usuário é preciso verificar se o e-mail já existe no banco de dados. Se o e-mail já existir, devemos retornar uma mensagem de erro. Se o e-mail não existir, devemos salvar o usuário no banco de dados.
Para implementar essa validação, seguiremos o seguinte fluxo:

### 16.1. Repositório: Buscar por E-mail

No `UsuarioRepository.php`, adicione um método que verifica se já existe um usuário com o e-mail informado.

```php
public function getUsuarioByEmail(string $email) {
    $sql = "SELECT id FROM usuarios WHERE email = :email";
    $stmt = $this->connection->prepare($sql);
    $stmt->bindValue(':email', $email);
    $stmt->execute();
    return $stmt->fetch(); // Retorna o usuário se encontrar, ou falso se não existir
}
```

### 16.2. Serviço: Validação da Regra de Negócio

No `UsuarioService.php`, modificamos o método `saveUsuario` para incluir a verificação antes de tentar salvar.

```php
public function saveUsuario(Usuario $usuario): bool {
    // 1. Verifica se o e-mail já está cadastrado
    $usuarioExistente = $this->repository->getUsuarioByEmail($usuario->getEmail());
    
    if ($usuarioExistente) {
        // Se o usuário existe, a regra de negócio impede o cadastro
        return false; 
    }
    
    // 2. Se não existir, prossegue com o salvamento
    return $this->repository->saveUsuario($usuario);
}
```

### 16.3. Controller: Feedback para o Usuário

No `UsuarioController.php`, verificamos o retorno do serviço. Se for `false`, podemos redirecionar com uma mensagem de erro.

```php
public function salvar() {
    $usuario = new Usuario(
        0, 
        $_POST['nomeUsuario'], 
        $_POST['email'], 
        $_POST['senha'], 
        $_POST['perfil']
    );

    if ($this->service->saveUsuario($usuario)) {
        $this->redirect(URL_BASE . '/usuarios');
    } else {
        // Aqui você poderia passar uma mensagem de erro para a view
        echo "Erro: Este e-mail já está cadastrado!";
    }
}
```
