## Introduction

L'objet `VkDescriptorSetLayout` que nous avons créé au chapitre précédent décrit les descripteurs que nous devons lier
pour les opérations de rendu. Dans ce chapitre nous allons créer les véritables sets de descripteurs, un pour chaque
`VkBuffer`, afin que nous puissions chacun les lier au descripteur UBO.

## Pool de descripteurs

Les sets de descipteurs ne peuvent pas être crées directement. Il faut les allouer depuis une pool, comme les command
buffers. Nous allons créer la fonction `createDescriptorPool` pour générer une pool de descripteurs.

```c++
void initVulkan() {
    ...
    createUniformBuffers();
    createDescriptorPool();
    ...
}

...

void createDescriptorPool() {

}
```

Nous devons d'abord indiquer les types de descripteurs et combien sont compris dans les sets. Nous utilisons pour cela
une structure du type `VkDescriptorPoolSize` :

```c++
VkDescriptorPoolSize poolSize = {};
poolSize.type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
poolSize.descriptorCount = static_cast<uint32_t>(swapChainImages.size());
```

Nous allons allouer un descripteur par frame. Cette structure doit maintenant être référencée dans la structure
principale `VkDescriptorPoolCreateInfo`.

```c++
VkDescriptorPoolCreateInfo poolInfo = {};
poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
poolInfo.poolSizeCount = 1;
poolInfo.pPoolSizes = &poolSize;
```

Nous devons aussi spécifier le nombre maximum de sets de descriptors que nous sommes susceptibles d'allouer.

```c++
poolInfo.maxSets = static_cast<uint32_t>(swapChainImages.size());;
```

La stucture possède un membre optionnel également présent pour les command pools. Il permet d'indiquer que les
sets peuvent être libérés indépendemment les uns des autres avec la valeur
`VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT`. Comme nous ne voulons pas toucher aux descripteurs pendant que le
programme s'exécute, nous n'avons pas besoin de l'utiliser. Vous pouvez laisser les `drapeaux` à leur valeur par défaut de `0`.

```c++
VkDescriptorPool descriptorPool;

...

if (vkCreateDescriptorPool(device, &poolInfo, nullptr, &descriptorPool) != VK_SUCCESS) {
    throw std::runtime_error("echec lors de la creation de la pool de descripteurs!");
}
```

Créez un nouveau membre donnée pour référencer la pool, puis appelez `vkCreateDescriptorPool`. La pool de descripteurs doit être détruit lorsque la chaîne de swap est recréée car cela dépend du nombre d'images :

```c++
void cleanupSwapChain() {
    ...

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        vkDestroyBuffer(device, uniformBuffers[i], nullptr);
        vkFreeMemory(device, uniformBuffersMemory[i], nullptr);
    }

    vkDestroyDescriptorPool(device, descriptorPool, nullptr);
}
```

Et recréé dans `recreateSwapChain` :

```c++
void recreateSwapChain() {
    ...

    createUniformBuffers();
    createDescriptorPool();
    createCommandBuffers();
}
```

## Set de descriptors

Nous pouvons maintenant allouer les sets de descipteurs. Créez pour cela la fonction `createDescriptorSets` :

```c++
void initVulkan() {
    ...
    createDescriptorPool();
    createDescriptorSets();
    ...
}

void recreateSwapChain() {
    ...
    createDescriptorPool();
    createDescriptorSets();
    ...
}

...

void createDescriptorSets() {

}
```

L'allocation de cette ressource passe par la création d'une structure de type `VkDescriptorSetAllocateInfo`. Vous devez
bien sûr y indiquer la pool d'où les allouer, de même que le nombre de sets à créer et l'organisation qu'ils doivent
suivre.

```c++
std::vector<VkDescriptorSetLayout> layouts(swapChainImages.size(), descriptorSetLayout);
VkDescriptorSetAllocateInfo allocInfo = {};
allocInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
allocInfo.descriptorPool = descriptorPool;
allocInfo.descriptorSetCount = static_cast<uint32_t>(swapChainImages.size());
allocInfo.pSetLayouts = layouts.data();
```

Dans notre cas nous allons créer autant de sets qu'il y a d'images dans la swap chain. Ils auront tous la même
organisation. Malheuresement nous devons copier la structure plusieurs fois car la fonction que nous allons utiliser
prend en argument un tableau, dont le contenu doit correspondre indice à indice aux objets à créer.

Ajoutez un membre donnée pour garder une référence aux sets, et allouez-les avec `vkAllocateDescriptorSets` :

```c++
VkDescriptorPool descriptorPool;
std::vector<VkDescriptorSet> descriptorSets;

...

descriptorSets.resize(swapChainImages.size());
if (vkAllocateDescriptorSets(device, &allocInfo, descriptorSets.data()) != VK_SUCCESS) {
    throw std::runtime_error("echec lors de l'allocation d'un set de descripteurs!");
}
```

Il n'est pas nécessaire de détruire les sets de descripteurs explicitement, car leur libération est induite par la
destruction de la pool. L'appel à `vkAllocateDescriptorSets` alloue donc tous les sets, chacun possédant un descripteur
de buffer uniform.

Nous avons créé les sets mais nous n'avons pas paramétré les descripteurs. Nous allons maintenant créer une boucle pour
rectifier ce problème :

```c++
for (size_t i = 0; i < swapChainImages.size(); i++) {

}
```

Les descripteurs référant à un buffer doivent être configurés avec une structure de type `VkDescriptorBufferInfo`. Elle
indique le buffer contenant les données, et où les données y sont stockées.

```c++
for (size_t i = 0; i < swapChainImages.size(); i++) {
    VkDescriptorBufferInfo bufferInfo = {};
    bufferInfo.buffer = uniformBuffers[i];
    bufferInfo.offset = 0;
    bufferInfo.range = sizeof(UniformBufferObject);
}
```

Nous allons utiliser tout le buffer, donc nous pourrions indiquer `VK_WHOLE_SIZE`. La configuration des
descripteurs est maintenant mise à jour avec la fonction `vkUpdateDescriptorSets`. Elle prend un tableau de
`VkWriteDescriptorSet` en paramètre.

```c++
VkWriteDescriptorSet descriptorWrite = {};
descriptorWrite.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrite.dstSet = descriptorSets[i];
descriptorWrite.dstBinding = 0;
descriptorWrite.dstArrayElement = 0;
```

Les deux premiers champs spécifient le set à mettre à jour et l'indice du binding auquel il correspond. Nous avons donné
à notre unique descripteur l'indice `0`. Souvenez-vous que les descripteurs peuvent être des tableaux ; nous devons donc
aussi indiquer le premier élément du tableau que nous voulons modifier. Nous n'en n'avons qu'un.

```c++
descriptorWrite.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
descriptorWrite.descriptorCount = 1;
```

Nous devons encore indiquer le type du descripteur. Il est possible de mettre à jour plusieurs descripteurs d'un même
type en même temps. La fonction commence à `dstArrayElement` et s'étend sur `descriptorCount` descripteurs.

```c++
descriptorWrite.pBufferInfo = &bufferInfo;
descriptorWrite.pImageInfo = nullptr; // Optionnel
descriptorWrite.pTexelBufferView = nullptr; // Optionnel
```

Le dernier champ que nous allons utiliser est `pBufferInfo`. Il permet de fournir `descriptorCount` structures qui
configureront les descripteurs. Les autres champs correspondent aux structures qui peuvent configurer des descripteurs
d'autres types. Ainsi il y aura `pImageInfo` pour les descripteurs liés aux images, et `pTexelBufferInfo` pour les
descripteurs liés aux buffer views.

```c++
vkUpdateDescriptorSets(device, 1, &descriptorWrite, 0, nullptr);
```

Les mises à jour sont appliquées quand nous appellons `vkUpdateDescriptorSets`. La fonction accepte deux tableaux, un de
`VkWriteDesciptorSets` et un de `VkCopyDescriptorSet`. Le second permet de copier des descripteurs.

## Utiliser des sets de descripteurs

Nous devons maintenant étendre `createCommandBuffers` pour qu'elle lie les sets de descripteurs aux descripteurs des
shaders avec la commande `cmdBindDescriptorSets`. Il faut invoquer cette commande dans la configuration des buffers de
commande avant l'appel à `vkCmdDrawIndexed`.

```c++
vkCmdBindDescriptorSets(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout, 0, 1, &descriptorSets[i], 0, nullptr);
vkCmdDrawIndexed(commandBuffers[i], static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```

Au contraire des buffers de vertices et d'indices, les sets de descripteurs ne sont pas spécifiques aux pipelines
graphiques. Nous devons donc spécifier que nous travaillons sur une pipeline graphique et non pas une pipeline de
calcul. Le troisième paramètre correspond à l'organisation des descripteurs. Viennent ensuite l'indice du premier
descripteur, la quantité à évaluer et bien sûr le set d'où ils proviennent. Nous y reviendrons. Les deux derniers
paramètres sont des décalages utilisés pour les descripteurs dynamiques. Nous y reviendrons aussi dans un futur
chapitre.

Si vous lanciez le programme vous verrez que rien ne s'affiche. Le problème est que l'inversion de la coordonnée Y dans
la matrice induit l'évaluation des vertices dans le sens des aiguilles d'une montre, alors que nous voudrions le
contraire. En effet, les systèmes actuels utilisent ce sens de rotation pour détermnier la face de devant. Le face de
derrière est ensuite simplement ignorée. C'est pourquoi notre géométrie n'est pas rendue. C'est le *backface culling*.
Changez le champ `frontface` de la structure `VkPipelineRasterizationStateCreateInfo` dans la fonction
`createGraphicsPipeline` de la manière suivante :

```c++
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE;
```

Maintenant vous devriez voir ceci en lançant votre programme :

![](/images/spinning_quad.png)

Le rectangle est maintenant un carré car la matrice de projection corrige son aspect. La fonction `updateUniformBuffer`
traite les redimensionnements d'écran, il n'est donc pas nécessaire de recréer les descripteurs dans
`recreateSwapChain`.

## Exigences d'alignement

Une chose que nous avons négligé jusqu'ici est comment exactement les données dans la structure C++ doit correspondre à la définition uniforme dans le shader. Il semble assez évident d'utiliser simplement les mêmes types dans les deux :

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

Mais ce n'est pas tout. Par exemple, essayez de modifier la structure et le shader pour qu'ils ressemblent à ceci :

```c++
struct UniformBufferObject {
    glm::vec2 foo;
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    vec2 foo;
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

Recompilez votre shader et votre programme et lancez-le et vous verrez que le carré coloré que vous avez travaillé jusqu'ici a disparu ! C'est parce que nous n'avons pas tenu compte des exigences d'alignement.

Vulkan s'attend à ce que les données de votre structure soient alignées en mémoire d'une manière spécifique, par exemple :

* Les mises à l'échelle doivent être alignées par N (= 4 octets avec des réel de 32 bits).
* Un `vec2` doit être aligné par 2N (= 8 octets)
* Un `vec3` ou `vec4` doit être aligné par 4N (= 16 octets)
* Une structure imbriquée doit être alignée par l'alignement de base de ses membres arrondi à un multiple de 16.
* Une matrice `mat4` doit avoir le même alignement qu'un `vec4`.

Vous trouverez la liste complète des exigences d'alignement dans [la spécification](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/chap14.html#interfaces-resources-layout).

Notre shader original avec seulement trois champs `mat4` répondait déjà aux exigences d'alignement. Comme chaque `mat4` a une taille de 4 x 4 x 4 = 64 octets, le `model` a un décalage de `0`, `vue` a un décalage de 64 et `proj` a un décalage de 128. Ce sont tous des multiples de 16 et c'est pourquoi ça a bien fonctionné.

La nouvelle structure commence avec un `vec2` qui n'a que 8 octets de taille et supprime donc tous les décalages. Maintenant, `model` a un décalage de `8`, `vue` un décalage de `72` et `proj` un décalage de `136`, dont aucun n'est un multiple de 16. Pour résoudre ce problème, nous pouvons utiliser le spécificateur [`d'alignement`](https://en.cppreference.com/w/cpp/language/alignas) introduit en C++11 :

```c++
struct UniformBufferObject {
    glm::vec2 foo;
    alignas(16) glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

Si vous compilez et exécutez à nouveau votre programme, vous devriez voir que le shader reçoit à nouveau correctement ses valeurs de matrice.

Heureusement, il existe un moyen de ne pas avoir à penser à ces exigences d'alignement la plupart du temps. Nous pouvons définir `GLM_FORCE_DEFAULT_ALIGNED_GENTYPES` juste avant d'inclure GLM :

```c++
#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEFAULT_ALIGNED_GENTYPES
#include <glm/glm.hpp>
```

Cela forcera GLM à utiliser une version de `vec2` et `mat4` qui a les exigences d'alignement déjà spécifiées pour nous. Si vous ajoutez cette définition, vous pouvez supprimer le spécificateur alignas et votre programme devrait toujours fonctionner.

Malheureusement, cette méthode peut tomber en panne si vous commencez à utiliser des structures imbriquées. Considérons la définition suivante dans le code C+++ :

```c++
struct Foo {
    glm::vec2 v;
};

struct UniformBufferObject {
    Foo f1;
    Foo f2;
};
```

Et la définition de shader suivante :

```glsl
struct Foo {
    vec2 v;
};

layout(binding = 0) uniform UniformBufferObject {
    Foo f1;
    Foo f2;
} ubo;
```

Dans ce cas, `f2` aura un décalage de `8` alors qu'il devrait avoir un décalage de `16` puisque c'est une structure imbriquée. Dans ce cas, vous devez spécifier l'alignement vous-même :

```c++
struct UniformBufferObject {
    Foo f1;
    alignas(16) Foo f2;
};
```

Ces pièges sont une bonne raison d'être toujours explicite sur l'alignement. De cette façon, vous ne serez pas pris au dépourvu par les symptômes étranges des erreurs d'alignement.

```c++
struct UniformBufferObject {
    alignas(16) glm::mat4 model;
    alignas(16) glm::mat4 view;
    alignas(16) glm::mat4 proj;
};
```

N'oubliez pas de recompiler votre shader après avoir supprimé le champ `foo`.

## Plusieurs sets de descripteurs

Comme on a pu le voir dans les en-têtes de certaines fonctions, il est possible de lier plusieurs sets de descripteurs
en même temps. Vous devez fournir une organisation pour chacun des sets pendant la mise en place de l'organisation de la
pipeline. Les shaders peuvent alors accéder aux descripteurs de la manière suivante :

```c++
layout(set = 0, binding = 0) uniform UniformBufferObject { ... }
```

Vous pouvez utiliser cette possibilité pour placer dans différents sets les descripteurs dépendant d'objets et les
descripteurs partagés. De cette manière vous éviter de relier une partie des descripteurs, ce qui peut être plus
performant.

[Code C++](/code/22_descriptor_sets.cpp) /
[Vertex shader](/code/21_shader_ubo.vert) /
[Fragment shader](/code/21_shader_ubo.frag)
