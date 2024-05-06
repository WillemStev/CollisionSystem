# Collision component gebruiken in een GameObject component (bvb FrogCollision in Frog)

in de hpp files van de collision component klassen staan de member variables die alle nodige info geven mbt collision van het bijhorend object vanboven

bvb bij de collision component FrogCollision houden we 6 member variables ```is_colliding_x_left```, ```is_colliding_x_right```, ```is_colliding_y_down```, ```is_colliding_y_up```, ```contact_enemy```, ```contact_enemy_projectile``` bij die info geven over het contact met het Terrain, Enemies en Enemy Projectiles

de declaratie van FrogCollision in de hpp file frog_collision.hpp ziet er zo uit

``` Cpp
class FrogCollision : public CollisionComponent {
  public:
    ...

    bool get_is_colliding_x_left();
    bool get_is_colliding_x_right();
    bool get_is_colliding_y_down();
    bool get_is_colliding_y_up();
    bool get_contact_enemy();
    bool get_contact_enemy_projectile();

  private:
    // variables that obtain the collision info of the Frog
    bool is_colliding_x_left;
    bool is_colliding_x_right;
    bool is_colliding_y_down;
    bool is_colliding_y_up;

    bool contact_enemy;
    bool contact_enemy_projectile;

    ...
};
```

in de klasse Frog kan je dan met de methods ```get_is_colliding_x_left```, ```get_is_colliding_x_right```, ```get_is_colliding_y_down```, ```get_is_colliding_y_up```, ```get_contact_enemy```, ```get_contact_enemy_projectile``` makkelijk kijken welke contacten er zijn

bvb de velocity_update functie in mijn FrogTest klasse ziet er zo uit --> puur om vlot alle gevallen uit te testen van Frog -- Terrain collision\
(frog_collision is de collision component FrogCollision gebruikt in FrogTest)

``` Cpp
void TestColFrog::velocity_update() {
    velocity_x_current = 0;
    velocity_y_current = 0;
    if (InputManager::get_instance().is_action_down("LEFT") && frog_collision->get_is_colliding_x_left() == false) {
        this->velocity_x_current = -100;
    }
    if (InputManager::get_instance().is_action_down("RIGHT") && frog_collision->get_is_colliding_x_right() == false) {
        this->velocity_x_current = 100;
    }
    if (InputManager::get_instance().is_action_down("DOWN") && frog_collision->get_is_colliding_y_down() == false) {
        this->velocity_y_current = 100;
    }
    if (InputManager::get_instance().is_action_down("JUMP") && frog_collision->get_is_colliding_y_up() == false) {
        this->velocity_y_current = -100;
    }
}
```

# mask principe
om héél snel te checken of 2 objecten met elkaar horen te colliden of niet wordt in de opgave voorgesteld om te werken met een Tag en een Mask

```app/utility/collider_tag.hpp``` is een bestand van de unief collider_tag.hpp --> MOVING_PLATFORM & TRAP & NOTHING zullen we wss niet gebruiken
``` Cpp
#ifndef COLLIDER_TAG_HPP
#define COLLIDER_TAG_HPP

enum ColliderTag {
    NOTHING = 0,
    TERRAIN = 1,
    CHARACTER = 2,
    ENEMY = 4,
    PICKUP = 8,
    PLAYER_PROJECTILE = 16,
    ENEMY_PROJECTILE = 32,
    TRAP = 64,
    MOVING_PLATFORM = 128
};

#endif
```
in deze enum krijgen Terrain, Character, Enemy, ... elk een Tag toegewezen, deze Tag is altijd een macht van 2 dat wil zeggen dat elk binair getal één 1tje bezit op een unieke plaats en al de rest is 0

``` Cpp
| Tag                | Decimal | Binary    |
|--------------------|---------|-----------|
| NOTHING            | 0       | 00000000  |
| TERRAIN            | 1       | 00000001  |
| CHARACTER          | 2       | 00000010  |
| ENEMY              | 4       | 00000100  |
| PICKUP             | 8       | 00001000  |
| PLAYER_PROJECTILE  | 16      | 00010000  |
| ENEMY_PROJECTILE   | 32      | 00100000  |
| TRAP               | 64      | 01000000  |
| MOVING_PLATFORM    | 128     | 10000000  |
```

nu ga je dit gebruiken om mbv logische bewerkingen op de tags te checken of 2 componenten moeten colliden of niet

bvb de Frog heeft ```CHARACTER``` als Tag en moet kunnen colliden met objecten die als Tag ```TERRAIN```, ```ENEMY```, ```PICKUP``` of ```ENEMY_PROJECTILE``` hebben

we maken voor de Frog een Mask\
`|` is de logische OR operator en `&` is de logische AND operator
``` Cpp
unsigned int mask = 001011010 = ColliderTag::ENEMY | ColliderTag::ENEMY_PROJECTILE | ColliderTag::PICKUP | ColliderTag::TERRAIN
```
nu checken we bvb of een objecten met Tag ```PLAYER_PROJECTILE``` en ```PICKUP``` horen te botsen tegen de Frog
``` Cpp
PLAYER_PROJECTILE & mask = 00000000 = 0
PICKUP & mask = 00001000 != 0
```
want ```PLAYER_PROJECTILE``` behoort niet tot de Tags in de Mask van Frog en ```PICKUP``` wel

dus als 
``` Cpp
Tag & Mask != 0 --> object met tag Tag hoort WEL te botsen tegen object met mask Mask
Tag & Mask == 0 --> object met tag Tag hoort NIET te botsen tegen object met mask Mask
```
\
\
\ 

# informatie overdracht tussen CollisionSystem en collision components (bvb FruitCollision)

TIP VAN SIEBE: als je niet weet vanwaar een functie komt dan doe je Ctrl klik op de functie en kom je in de cpp file terecht waar hij gedefinieerd staat

informatie tussen CollisionSystem en collision components wordt doorgegeven via de methods in de klasse CollisionComponent waar alle collision components van overerven

bvb in de constructor van FruitCollision doe ik
``` Cpp
this->set_tag(ColliderTag::PICKUP); // functie van CollisionComponent
```
set_tag is een functie van CollisionComponent waar FruitCollision toegang toe heeft door overerving; de tag ColliderTag::PICKUP wordt dan opgeslaan in de klasse CollisionComponent

in het CollisionSystem wordt over alle klassen CollisionComponent geïtereerd, daar kan ik dan te weten komen welke tag de CollisionComponent klasse heeft door de functie get_tag van CollisionComponent op te roepen
``` Cpp
c_vector[j]->get_tag() // c_vector[j] is een CollisionComponent object
```
\
\
\ 
# CollisionSystem
## 1e for loop
* vul een vector c_vector met alle huidige collision components in de game
* resize de vector collision_info naar 0 van elke collision component (vector collision_info leg ik later uit) --> voordeel van vector.resize(0) tov vector.clear(): er moet niet elke update functie opnieuw memory gealloceerd en gedealloceerd worden

elk object bevat een collision component (Frog, Fruit, Kogel); voor het Terrain zal elke tile van het Terrain apart toegevoegd moeten worden als collision component

## 2e for loop

we vergelijken alle componenten uit c_vector met elkaar maar we vermijden dubbels

```
c_vector = {1, 2, 3, 4, 5, 6, 7, 8, 9, … } --> allemaal collision components

vergelijk collision components zonder dubbels
alle collision pairs:
(1, 2), (1, 3), (1, 4), (1, 5), (1, 6), ...
        (2, 3), (2, 4), (2, 5), (2, 6), ...
                (3, 4), (3, 5), (3, 6), ...
```

de 1e component in het collision pair is component i de 2e component is component j\
dus collision pair (i, j)

### 1e if statement
gebruik de mask van collision component i en de tag van collision component j om te zien of collision pair (i, j) hoort te botsen of niet
``` Cpp
if ((c_vector[i]->get_mask() & c_vector[j]->get_tag()) != 0) 
```
### 2e if statement
check of de collision boxes van component i en j in dezelde map box zitten of niet door te kijken of hun vector map_boxes eenzelfde element bezit

bvb in de update functie van de Frog die een collision component ```FrogCollision* frog_collision``` bezit zullen de coordinaten vd Frog elke keer doorgegeven worden via ```frog_collision->update_frog_collision(Vector2 position)```; daar zullen de map boxes dan ook geupdate worden

in de update_frog_collision functie van FrogCollision gebruik je dan de functies set_collision_box_coords en set_map_boxes van CollisionComponent; in CollisionSystem roep je dan get_map_boxes en get_collision_box_coords op
![image](https://github.com/WillemStev/CollisionSystem/assets/153719651/e87d10de-bdd8-4a2a-a513-f98e4427bdb6)
![image](https://github.com/WillemStev/CollisionSystem/assets/153719651/e2dbac3f-0e26-4ba9-82df-2a9be16e3ac4)

\
\

### vector collision_info

als beide if statements voldaan zijn heb je goeie kans dat er een botsing is

dan ga je via de add_collision_info functie van CollisionComponent

* een tuple ```(ColliderTag tag_component_j, vector<int> collision_box_coords_i, vector<int> collision_box_coords_j, (char)'i')```  toevoegen aan de vector collision_info van component i
  
* een tuple ```(ColliderTag tag_component_i, vector<int> collision_box_coords_i, vector<int> collision_box_coords_j, (char)'j')```  toevoegen aan de vector collision_info van component j
  
collison_info is een member variable van CollisionComponent

dus component i krijgt via deze tuple de volgende info:
* de tag_component_j laat weten wat de ColliderTag is van het andere object
* collision_box_coords_i & collision_box_coords_j geven de coordinaten van collision boxes van componenten i en j
* de char 'i' laat component i weten dat hij component i is --> dit is belangrijk wanneer de volgorde van de botsing van de belang is

hetzelfde voor j

\
\

## functie collision_status()

de functie collision_status() is net zoals de render() en set_next_transition() functie uit RenderingSystem en AnimationSystem

in deze functie gebeurt de collision detection tussen collision pairs (i, j)

het verloop van de functie ziet er als volgt uit

in FrogCollision
``` Cpp
void FrogCollision::collision_status() {

    ...

    // iterate over all tuples in the vector collision_info
    auto collision_info = this->get_collision_info();
    for (auto it = collision_info.begin(); it != collision_info.end(); ++it) {

        tuple tuple_info = *it;
        ColliderTag tag_OtherCollidingObject = get<0>(tuple_info);
        vector<int> collision_box_coords_i = get<1>(tuple_info);
        vector<int> collision_box_coords_j = get<2>(tuple_info);
        char i_or_j = get<3>(tuple_info);

        int x1_i = collision_box_coords_i[0], x2_i = collision_box_coords_i[1], y1_i = collision_box_coords_i[2], y2_i = collision_box_coords_i[3];
        int x1_j = collision_box_coords_j[0], x2_j = collision_box_coords_j[1], y1_j = collision_box_coords_j[2], y2_j = collision_box_coords_j[3];

        if (tag_OtherCollidingObject == ENEMY) {
            // perform collision detection and decide whether this->contact_to_enemy = true/false
        }
        else if (tag_OtherCollidingObject == ENEMY_PROJECTILE) {
            // perform collision detection and decide whether this->contact_to_enemy_projectile = true/false
        }
        else if (tag_OtherCollidingObject == TERRAIN) {
            // perform collision detection and decide whether this->is_colliding_x_left = true/false; this->is_colliding_x_right = true/false; this->is_colliding_y_down = true/false; this->is_colliding_y_up = true/false
        }

        ...

}
```

in de FrogCollision zal je de Tag PICKUP niet checken; dit doe je in de collision_status() functie van FruitCollision

in FruitCollision
``` Cpp
void FruitCollision::collision_status() {
        ...
        if (tag_OtherCollidingObject == CHARACTER) {
            // perform collision detection and decide whether this->is_captured = true/false
        }
        ...
}
```

# Collision detection
## locatie van componenten i en j ten opzichte van elkaar NIET belangrijk
als de plaats van component i tov component j niet boeit dan is de voorwaarde voor collision heel simpel: de 2 collision boxes moeten gewoon overlappen

dit is zo in elk collision pair behalve Frog -- Terrain, Enemy -- Terrain

als 2 collision boxes overlappen is deze voorwaarde voldaan

```x1_i < x2_j && x1_j < x2_i) && (y1_i < y2_j && y1_j < y2_i```


![image](https://github.com/WillemStev/CollisionSystem/assets/153719651/dc168465-4975-49cc-b130-ece68fefeef0)

## locatie van componenten i en j ten opzichte van elkaar WEL belangrijk

het gevallenonderscheid van de collision detection tussen Frog en Terrain is een beetje een merde om hier uit te leggen; als je het echt wil verstaan zullen we best samen door de code gaan

we houden in FrogCollision de variabelen ```is_colliding_x_left```, ```is_colliding_x_right```, ```is_colliding_y_down```, ```is_colliding_y_up``` bij

bvb hier ```is_colliding_x_right``` geldt want de Terrain tile botst tegen de rechterkant vd Frog; de rechterkant van de Frog covert minstens 32 pixels aan contactoppervlak met tiles van terrain ==> er is full contact

![image](https://github.com/WillemStev/CollisionSystem/assets/153719651/84b81410-3590-49b8-b347-d7a3b8295723)


bij het collision pair Frog -- Terrain moet je opletten

Hier mag je bvb geen zijlings contact veronderstellen maar wel neerwaarts contact. We zullen in het begin vd collision detection van Frog -- Terrain dus enkel ```is_colliding_x_left```, ```is_colliding_x_right```, ```is_colliding_y_down```, ```is_colliding_y_up``` true veronderstellen als er full contact is

![image](https://github.com/WillemStev/CollisionSystem/assets/153719651/232056d2-c941-4220-8a9c-d2708505b513)

er is ook een speciaal geval mogelijk: geen full contact in de 4 richtingen maar wel contact --> in dit geval zal er SLECHTS 1 TERRAIN TILE in contact komen met de Frog

in dat geval werken we met een treshold: als het kleinste verschil in y-coordinaten ```min(y2_i - y1_j, y2_j - y1_i) > treshold``` dan is er teveel overlap in de y-richting om te kunnen zeggen dat er neerwaarts contact is en moeten we zijlings contact veronderstellen

op de figuur is ```y2_i - y1_j < treshold``` ==> niet te veel overlap in de y-richting dus kunnen we neerwaarts contact veronderstellen, als de Frog verticaal naar beneden in deze positie landt is het ook logisch dat we neerwaarts contact veronderstellen; als de collision detection performant gebeurt zal de Frog bij zijn landing niet door de treshold barriere breken

als ```min(y2_i - y1_j, y2_j - y1_i) <= treshold``` dan veronderstellen we neerwaarts contact; de waarde van treshold is wat gokken

als de strategie met de treshold niet realistisch overkomt in de game moeten we hem gwn wat aanpassen

![image](https://github.com/WillemStev/CollisionSystem/assets/153719651/9b2cf442-c650-4a98-9dc6-67b45ca801c8)

\
\
## Correctie vd positie

bij collision Frog -- Terrain en Enemy -- Terrain zakken de Frog en Enemy wat in het terrain, we moeten dan de coordinaten van Frog en Enemy aanpassen zodat hun collision boxes raken aan de collision boxes van Terrain
