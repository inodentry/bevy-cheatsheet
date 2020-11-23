# Folha de dicas para Bevy Game Engine

Folha de dicas concisas para mostrar as sintaxes exatas de recursos comuns e padrões de programação em [Bevy Game Engine](https://github.com/bevyengine/bevy).

Abrange principalmente a versão atual (v0.3), mas as alterações na versão mais recente (git) também são notados quando aplicáveis.

Este documento é mantido com grande esforço. Algumas informações podem estar desatualizadas.

Ajude a melhorá-lo e mantê-lo atualizado, contribuindo em [GitHub](https://github.com/jamadazi/bevy-cheatsheet).

Se você gosta disso, você também deve dar uma olhada em [Livro de receitas para Bevy Game Engine](https://github.com/jamadazi/bevy-cookbook).

## Entidades (Entities)

Conceitualmente, um "objeto" no jogo.

Tecnicamente, apenas um ID para acessar os tipos de dados (data-type): `Entidade`.

## Componentes (Components)

Componentes são dados por entidade.

Definido como estruturas simples de Rust. Acessado por meio de consultas `Query`.

```rust
struct MeuComponente {
    dado_um: u32,
    dado_dois: f32,
}
```

Estruturas vazias (tamanho zero) podem ser usadas como "componentes de marcador" para identificar/marcar entidades (útil em consultas `With`/`Without`, veja abaixo).

```rust
struct MarcadorVazio;
```

Os componentes são identificados por seu tipo; uma entidade não pode ter mais de uma instância do mesmo tipo de componente. Se precisar de mais, você pode envolvê-los em estruturas de tipo novo para criar tipos exclusivos.

## Pacotes de Componentes (Component Bundles)

Os componentes podem ser agrupados em pacotes para facilitar a geração de uma entidade com um conjunto comum de componentes.

```rust
#[derive(Bundle)]
struct MeuComponenteABC {
    a: ComponenteA,
    b: ComponenteB,
    c: ComponenteC,
}
```

## Recursos (Resources)

Semelhante a "variáveis globais", usados para conter dados independentes de entidades.

Definido como estruturas simples de Rust. Acessado usando os parâmetros `Res`/`ResMut`.

Os recursos são identificados por seu tipo; não pode haver mais de um recurso do mesmo tipo. Se precisar de mais, você pode envolvê-los em estruturas de tipo novo para criar tipos exclusivos.

## Inicialização de recursos (Resource Initialization)

Você pode inicializar seus recursos com `FromResources`:

```rust
struct MeuRecursoSofisticado { /* stuff */ }

impl FromResources for MeuRecursoSofisticado {
    fn from_resources(recursos: &Resources) -> Self {
        // você tem acesso total a quaisquer outros recursos de que precisa aqui,
        // você pode até transformá-los:
        let mut x = recursos.get_mut::<MeuOutroRecurso>().unwrap();
        x.do_mut_stuff();

        MeuRecursoSofisticado { /* stuff */ }
    }
}
```

Se você não precisa de acesso a outros recursos, você pode implementar ou derivar `Default`:

```rust
#[derive(Default)]
struct RecursoSimples {
    flutuante: f32,
    opcao: Option<f32>,
    vetor: Vec<f32>,
}
```

Você também pode construir seu valor e adicioná-lo por meio do construtor `App`.

Você também pode inserir recursos de um sistema de inicialização, usando `Commands`.

## Sistemas (Systems)

Os sistemas são funções regulares de Rust que recebem tipos especiais como argumentos.

Usado para operar no estado do jogo acessando componentes e recursos.

Eles precisam ser registrados no momento da criação do `App`.

*Nota de versão (v0.3)*: Bevy requer primeiro uma ordem específica para os argumentos `fn`:` Comandos`, seguido por recursos e consultas por último.

*Nota de versão (git)*: O requisito de pedido foi removido. Os parâmetros do sistema podem estar em qualquer ordem agora.

## Consultas (Queries)

As consultas permitem que você opere em entidades com um determinado conjunto de componentes.

Você pode obter acesso mutável/imutável a componentes específicos.

Você pode filtrar com base na presença/ausência de um tipo de componente, usando `With<T, ...>`/`Without<T, ...>`.

Você também pode usar `Option` como um adaptador para obter um componente se ele existir.

```rust
fn meu_sistema_complexo(
    // os recursos devem vir antes das consultas
    recurso1: Res<MeuRecurso>,
    mut recurso2: ResMut<MeuOutroRecurso>,

    // consultas devem ser definidas como mutáveis (`mut`), mesmo se os próprios componentes não forem
    mut consulta1: Query<(&mut ComponenteA, &ComponenteB)>,

    // use `Entity` para obter o ID da entidade
    mut consulta2: Query<(Entity, &mut ComponenteC)>,

    // só obter entidades que tenham um componente `Foo` (Note Query<With<Foo, ...>>)
    mut consulta3: Query<With<Foo, (&Bar, &mut Baz)>>,

    // só obter entidades sem um componente `Abc` (Note Query<Without<Abc, ...>>)
    mut consulta4: Query<Without<Abc, (&mut Def, Entity)>>,

    // opcionalmente, obtenha acesso ao `Cde` se ele existir em entidades que possuem `Bcd`
    mut consulta5: Query<(&Bcd, Option<&mut Cde>)>,
) {
    // iterar em todas as entidades correspondentes
    for (mut a, b) in consulta1.iter_mut() {
        // `a` é um tipo de wrapper especial (`Mut<T>`) que se refere ao componente real
        *a = ComponenteA::new();
    }

    // ou consultar apenas uma entidade específica
    if let Ok((bar, mut baz)) = consulta3.get(minha_entidade) {
        // faz coisas de Bar e Baz com `minha_entidade`
    }

    // ou apenas um componente de uma entidade específica
    if let Ok(mut c) = consulta2.get_component_mut::<ComponenteC>(minha_entidade) {
        // fazer coisas com ComponenteC
    }
}
```

*Nota de versão (git)*: A sintaxe para `With` e `Without` mudou. Veja [Query Filters](#query-filters).

## Consultas conflitantes (Conflicting queries)

Por razões de segurança, um sistema não pode ter várias consultas com conflitos de mutabilidade nos mesmos componentes:

```rust
// não permitido (não conseguirá compilar): porque ambas as consultas querem `&mut ComponenteA`
fn meu_sistema(
    mut consulta1: Query<(&mut ComponenteA, &ComponenteB)>,
    mut consulta2: Query<(&mut ComponenteA, &ComponenteB)>
)
```

A solução é envolvê-los em um `QuerySet`:

```rust
fn meu_sistema(
    mut consulta_queryset: QuerySet<(
        Query<(&mut ComponenteA, &ComponenteB)>,
        Query<(&mut ComponenteA, &ComponenteB)>
    )>
) {
    for a in consulta.q0_mut().iter_mut() {
    }

    for (a, b) in consulta.q1_mut().iter_mut() {
    }
}
```

Isso garante que apenas uma das consultas conflitantes possa ser usada ao mesmo tempo.

## Detecção de mudança (Change detection)

*Nota de versão (git)*: esta sintaxe foi substituída por [Query Filters](#filtros-de-consulta-(query-filters)).

Consultas especiais podem ser usadas para verificar se os componentes foram modificados por outros sistemas neste quadro (frame).

Observe que os vários tipos de detecção de alteração permitem apenas acesso imutável.

```rust
fn sistema_de_rastrear_mudanca(
    // apenas entidades cujo componente `Qux` foi modificado por outro sistema neste quadro
    mut consulta1: Query<(Mutated<Qux>, &OutroDado)>,
    // apenas entidades cujo componente `Thing` foi adicionado recentemente
    mut consulta2: Query<(Added<Thing>, &OutroDado)>,
    // adicionado ou modificado
    mut consulta3: Query<(Changed<Stuff>, &OutroDado)>,
) {
    // para detectar remoções, use este método (não há tipo de consulta para removidos)
    for entidade in consulta3.removed::<OutroComponente>() {
        // fazer algo com cada entidade `consulta3` que perdeu seu `OutroComponente`
    }
}
```

Para observar as mudanças em vários componentes:

```rust
// todos eles (lógica E = AND)
Query<(Mutated<Abc>, Mutated<Bcd>)>
// qualquer um deles (lógica OU = OR)
Query<Or<(Mutated<Cde>, Mutated<Def>)>>
// complexo
Query<(Mutated<Foo>, Or<(Mutated<Xy>, Mutated<Yx>)>)>
```

Os recursos também têm detecção de mudança básica usando `ChangedRes<T>`:

```rust
// todo o sistema só funciona se o recurso foi alterado
fn sistema_de_mudanca_de_recurso(my_res: ChangedRes<MeuRecurso>) {
    eprintln!("Resource changed to {:?}", *my_res);
}
```

## Filtros de Consulta (Query Filters)

*Nota de versão (v0.3)*: esta sintaxe não está disponível. Veja [Change detection](#change-detection) e [Queries](#queries).

O tipo de consulta é, na verdade, `Query<C, F = ()>`, onde `C` são os componentes que você deseja acessar e `F` é um filtro, para aplicar restrições adicionais para selecionar quais entidades devem corresponder.

Use tuplas (tuples) para acessar vários componentes e aplicar vários filtros.

Exemplos:

```rust
// acesso mutável a Foo, mas apenas se ele mudou
Query<&mut Foo, Changed<Foo>>

// acessar Bar e Baz, mas apenas se a entidade não tiver um Foo e seu Qux foi alterado
Query<(&Bar, &Baz), (Without<Foo>, Mutated<Qux>)>

// acessar Abc mutável se a entidade tiver Def ou Fed
Query<&mut Abc, Or<With<Def>, With<Fed>>>

// consulta regular sem filtro (o argumento do filtro é opcional)
Query<(&DataA, &mut DataB, &DataC)>
```

## Comandos (Commands)

Spawn e despawn entidades, adicionar/remover componentes, inserir recursos, usando `Commands`.

*Nota de versão (v0.3)*: Deve ser o primeiro argumento do `fn`.

```rust
fn sistema_de_gerenciamento(
    mut comando: Commands, data: Res<MeuRecurso>,
    mut consulta: Query<(Entity, &Stuff)>
) {
    // gerar uma entidade; leva um pacote de componentes
    comando.spawn(MeusComponentesAbc {
        a: ComponenteA::new(),
        b: ComponenteB::new(),
        c: ComponenteC::new()
    }).with(ComponenteExtra::default()); // adiciona um ExtraComponent

    // tuplas também são pacotes de componentes
    comando.spawn((Foo::default(), Bar::default(), data.stuff.clone()));

    // para obter o id de entidade da última entidade gerada:
    let my_e = comando.current_entity().unwrap();

    for (e, s) in &mut consulta.iter() {
        if s.is_bad() {
            comando.despawn(e); // despawnar a entidade
        }
    }
}
```

*Nota de versão (v0.3)*: O tipo de parâmetro é `mut cmd: Commands`, como no exemplo acima.

*Nota de versão (git)*: O tipo de parâmetro mudou, agora você tem duas opções:

* `cmd: &mut Commands`: acessa `Commands` diretamente, mas previne que seu sistema rode em paralelo com outros sistemas que precisam dele.
* `cmd: Arc<Mutex<Commands>>`: acessar `Commands` atrás de um `parking_lot::Mutex`. Isso permite que seus sistemas funcionem em paralelo, mas tem alguns problemas:
  * Você pode causar um deadlock se bloquear o mutex novamente em um sistema que já possui o bloqueio.
  * Os métodos `Commands` que operam na entidade atual, como `with` e `current_entity`, quebram se você liberar o bloqueio entre a chamada para `spawn` e chamá-los. (bevyengine/bevy#795)

## Recursos Locais (Local Resources)

Você pode ter dados por sistema usando `Local<T>`.

```rust
fn meu_sistema(data: Local<MeuDado>) { }
```

`T` deve implementar `Default` ou `FromResources`, pois precisa ser inicializado automaticamente.

Se o mesmo tipo for usado em vários sistemas, cada um deles obterá sua própria instância.

## Eventos (Events)

Envie notificações entre sistemas. Acessado usando um recurso `Events<T>`.

Para receber, você precisa de um `EventReader<T>`. Ele rastreia os eventos processados. É bom tê-lo como um recurso `Local`.

Os eventos não persistem. Se os receptores não lidarem com todos os quadros, eles serão perdidos.

```rust
struct MeuEvento;

fn meu_sistema_de_receber(eventos: Res<Events<MeuEvento>>, mut leitor: Local<EventReader<MeuEvento>>) {
    for ev in leitor.iter(&eventos) {
        // faça algo com `ev`
    }
}

fn meu_sistema_de_envio(mut eventos: ResMut<Events<MeuEvento>>) {
    eventos.send(MeuEvento);
}
```

Os tipos de eventos devem ser registrados ao construir o `App`.

## Inicialização do aplicativo - função principal (App Initialization - main function)

Para entrar no tempo de execução do Bevy, você deve construir um `App`, registrar quaisquer plug-ins, eventos, recursos e sistemas que você usa e chamar `run()`.

```rust
fn main() {
    App::build()
        .add_plugins(DefaultPlugins)
        .add_event::<MeuEvento>()
        // construir um valor para o recurso MeuRecurso1
        .add_resource::<MeuRecurso1>(MeuRecurso1::new())
        // se implementa `Default` ou` FromResources`
        .init_resource::<MeuRecurso2>()
        // é executado uma vez na inicialização, antes dos sistemas normais
        .add_startup_system(meu_sistema_de_setup.system())
        // executa em cada quadro
        .add_system(meu_sistema_de_game.system())
        .run();
}
```

## Plugins

Conforme seu aplicativo cresce, pode ser útil torná-lo mais modular. Você pode dividi-lo em plugins:

```rust
struct MeuPlugin;

impl Plugin for MeuPlugin {
    fn build(&self, app: &mut AppBuilder) {
        // adicione coisas ao seu aplicativo aqui
        app
            .add_event::<MeuEvento>()
            .init_resource::<MeuRecurso>()
            .add_system(meu_sistema.system());
    }
}

fn main() {
    App::build()
        .add_plugins(DefaultPlugins)
        .add_plugin::<MeuPlugin>();
}
```

## Ativos (Assets)

Os recursos `Assets<T>` armazenam os dados reais. Eles são indexados usando `Handle<T>`s, que são IDs leves para recursos individuais carregados. O recurso `AssetServer` lida com o carregamento de ativos no motor.

```rust
struct MeuComponente {
    asset: Handle<Texture>,
}

fn sistema_do_game(storage: Res<Assets<Texture>>, data: &MeuComponente) {
    // ativos (assets) são carregados em segundo plano
    // pode ser `None` se ainda não foi concluído
    if let Some(asset) = storage.get(&data.asset) {
        // fazer algo com os dados do ativo (asset)
    }
}

fn setup(comandos: Commands, server: Res<AssetServer>) {
    let handle = server.load("texture.png");
    comandos.spawn(
        MeuComponente {
            asset: handle
        },
    );
}
```

Você também pode adicionar valores diretamente ao armazenamento `Assets<T>` e ignorar o `AssetServer`, se você os carregou/gerou de outra maneira.

## Entidades Hierárquicas (Hierarchical Entities)

Entities can be nested into parent/child hierarchies.
As entidades podem ser aninhadas em hierarquias pai/filho.

```rust
// criar entidade com um filho
let e = comandos
    .spawn(MeusComponentesPrincipais::default())
    .with_children(|parent| {
        parent.spawn(MeusComponentesFilhos::default());
    });

// entidade despawn junto com seus filhos
comandos.despawn_recursive(entity);
```

A entidade pai terá um componente `Children`, que contém os ids de entidade dos filhos.

Os filhos terão um componente `Parent`, que contém o id da entidade do pai.

## Transformações Hierárquicas (Hierarchical Transforms)

Para usar transformações com entidades hierárquicas, você deve adicionar um componente `GlobalTransform` e um componente `Transform`.

The `GlobalTransform` will be managed by bevy internally.
O `GlobalTransform` será gerenciado internamente pelo Bevy.

O `Transform` é sua transformação local. Principalmente para os filhos, mas é relativo ao pai.

## Recursos integrados úteis (Useful built-in resources)

 - `AssetServer`: use para carregar ativos do disco
 - `Input<KeyCode>`, `Input<MouseButton>`: para verificar se as teclas/botões estão pressionados
 - `Time`: tempo delta do quadro e tempo de execução geral
 - `Windows`: parâmetros das janelas abertas, como dimensões...

## Recursos de configuração (Configuration resources)

Esses recursos integrados podem ser adicionados ao construir seu `App`, para configurar várias coisas:

 - `ClearColor`: definir a cor clara (cor de fundo)
 - `Msaa`: habilitar Multi-Sample Anti-Aliasing (MSAA)
 - `WindowDescriptor`: alterar os parâmetros da janela principal

Observe que alguns deles devem ser adicionados antes de `DefaultPlugins` para que seus valores tenham efeito.

## Useful built-in events

 - Dispositivos de entrada: `KeyboardInput`, `CursorMoved`, `MouseMotion`, `MouseButtonInput`, `MouseWheel`.

## Pacotes de componentes integrados úteis (Useful built-in component bundles)

 - `Camera2dComponents`: câmera ortográfica 2D
 - `Camera3dComponents`: câmera 3D em perspectiva
 - `UiCameraComponents`: câmera para exibir a IU
 - `SpriteComponents`, `SpriteSheetComponents`: sprites 2D
 - `MeshComponents`: malhas 3D
 - `LightComponents`: luzes 3D
 - `TextComponents`: texto da IU
 - `ButtonComponents`: botão da IU

## Componentes integrados úteis (Useful built-in components)

 - `Draw`: estado de renderização; defina a visibilidade e habilite a transparência aqui
 - `Transform`: a transformação (posição, rotação, escala) de um objeto no mundo do jogo
 - `TextureAtlasSprite`: o id do sprite em uma spritesheet
 - `Timer`: detecta quando um intervalo de tempo passou; pode estar se repetindo

## Tipos de ativos integrados úteis (Useful built-in asset types)

 - `Font`: fonte para renderizar o texto
 - `Texture`: dados da imagem
 - `TextureAtlas`: spritesheet
 - `Mesh`: geometria 3D

## Sistemas integrados úteis (Useful built-in systems)

Sistemas que vêm com o Bevy, mas não são ativados por padrão. Adicione-os ao seu aplicativo, se quiser.

 - `bevy::input::system::exit_on_esc_system`: fecha o aplicativo ao pressionar a tecla Esc.

## Truques de sintaxe (Syntax tricks)

Para contornar as limitações no número de parâmetros, eles podem ser arbitrariamente aninhados em tuplas:

```rust
fn muitos_recursos(
    (recurso1, mut recurso11): (Res<Recurso1>, ResMut<Recurso11>),
    (recurso2, recurso12): (Res<Recurso2>, Res<Recurso12>),
    recurso3: Res<Recurso3>,
    mut recurso4: ResMut<Recurso4>,
    recurso5: Res<Recurso5>,
    recurso6: Res<Recurso6>,
    mut recurso7: ResMut<Recurso7>,
    recurso8: Res<Recurso8>,
    recurso9: Res<Recurso9>,
    recurso10: Res<Recurso10>, // o número máximo de parâmetros de recursos é 10
    // mut recurso11: ResMut<Recurso11>, // isso causaria um erro de compilação
) { /* ... */ }
```