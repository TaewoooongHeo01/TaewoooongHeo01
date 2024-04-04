---
layout: post
title: React - ContextAPI 를 통한 컴포넌트 간 데이터 공유
category: Others
excerpt: "컴포넌트 간 데이터 공유를 위해선 props 를 사용한다고 알고 있어 어떻게 구현할 지 생각해봤다. 
기본적으로 props 는 부모-자식 관계의 컴포넌트 간 데이터를 공유할 때 사용한다. 하지만 위 기능을 props 로 구현하기엔 너무 불편한 것 같다. props 가 너무 많고, 만약 추후에 props 가 추가된다면 수많은 코드의 수정이 필요할 것이다. 그래서 나는 다른 방법을 찾아봤고, ContextAPI 를 발견했다."
---

아래의 프로젝트와 이어집니다.<br>
-> [Jackson 직렬화 순환참조 에러 해결하기](https://taewoooongheo01.github.io/TaewoooongHeo01/spring/2024/03/15/jackson/)

간략하게 상황을 설명하자면, 음식 재료정보 리스트를 BE 에서 FE 로 전달받았다. 

API 를 통해 전달받은 음식 재료 정보 JSON 을 FE 에 랜더링한 뒤, 검색기능까지 구현한 상태이다. 

![](https://i.imgur.com/nlEE8Iu.gif)

이제 내가 구현할 기능은 재료를 클릭하면 오른쪽 "재료 선택" 박스에 해당 재료가 담기는 기능이다. 

재료 정보를 받아와 검색기능을 구현하는 컴포넌트는 아래와 같이 만들었다(왼쪽 박스). 

`IGDScroll` 이 재료 정보로 스크롤을 구현한 컴포넌트이다(참고로 tailwind, shadcn ui 를 사용했다)

{% highlight JSX%}
import React from "react";
import "../App.css";
import { useState, useEffect } from "react";
import { ScrollArea } from "./ui/scroll-area"
import { Separator } from "./ui/separator"

function IGDScroll() {

   const [ingredients, setIngredients] = useState([]);
   const [searchResults, setSearchResults] = useState([]);

   //검색기능
   const handleSearch = (e) => {
      let keyword = e.target.value;
      if (keyword.trim() === '') { // 검색어가 없을 경우 모든 재료를 표시
         setSearchResults(ingredients);
      } else {
         const filteredData = ingredients.filter(data =>
   data.ingredientName.toLowerCase().includes(keyword.toLowerCase())
         );
         setSearchResults(filteredData); // 검색 결과 업데이트
      }
   };

   useEffect(() => {
   // 데이터를 가져오기
      const fetchData = async () => {
         try {
            const response = await fetch('http://localhost:8080/');
            if (!response.ok) {
               throw new Error('Data could not be fetched!');
            } else {
               const data = await response.json();
               setIngredients(data); // 상태 업데이트
               setSearchResults(data); // 초기 검색 결과 설정
            }
         } catch (error) {
            console.error("Error fetching data: ", error);
         }
      };
   
      fetchData(); 
   }, []); 

   return (
      <>
         <input type="text" onChange={handleSearch} placeholder="재료검색" className="{% raw %}css code...{% endraw %}"/>
         {% raw %}<ScrollArea className="w-64 rounded-md pl-4 pr-4 pb-4" style={{height: "90%"}}>{% endraw %}
            <div className="p-1">
               {searchResults.map((ingredient, index) => (
                  <React.Fragment key={index}>
                     <div className="text-sm">
                        {ingredient.ingredientName}
                     </div>
                     <Separator className="my-2" />
                  </React.Fragment>
               ))}
            </div>
         </ScrollArea>
      </>
   );
}

{% endhighlight %}

아래는 선택된 재료를 담을 `SelectedList` 컴포넌트이다(오른쪽 박스). 
{% highlight JSX%}
import React from "react";
import "../App.css";
import { ScrollArea } from "./ui/scroll-area"
import { Separator } from "./ui/separator"

function SelectedList() {
   return (
      <>
         <h4 className="mb-4 pl-4 pt-4 text-sm font-medium leading-none">재료 선택</h4>
         {% raw %}<ScrollArea className="w-64 rounded-md pl-4 pr-4 pb-4" style={{height: "90%"}}>{% endraw %}
            <div className="p-1">
            </div>
         </ScrollArea>
      </>
   )
}
{% endhighlight %}

# 클릭 이벤트 추가
나는 일단 재료 클릭 시, 해당 재료를 콘솔에 출력해봤다. 
{% highlight JSX%}
{% raw %}<div className="text-sm" onClick={() => {console.log(ingredient.ingredientName)}} style={{"cursor": "pointer"}}>{% endraw %}
   {ingredient.ingredientName}
</div>
{% endhighlight %}

`onClick` 이벤트를 통해 간단히 구현했다. 
# 데이터 공유 방법 구상
사실 문제는 지금부터였다. 현재 내가 구현해야 하는 기능은 클릭 시 해당 재료 이름이 `SelectedList` 컴포넌트로 정보가 넘어가야 한다. 

컴포넌트 간 데이터 공유를 위해선 `props` 를 사용한다고 알고 있어 어떻게 구현할 지 생각해봤다. 

기본적으로 props 는 부모-자식 관계의 컴포넌트 간 데이터를 공유할 때 사용한다. 위 기능을 props 로 구현하기 위해 생각한 방안은 다음과 같다. 
- `IGDScroll` , `SelectedList` 컴포넌트를 감싸는 부모 컴포넌트를 만듦
- IGDScroll 에서 재료 정보 클릭 시 props 를 통해 부모 컴포넌트로 보낸다.
- 부모 컴포넌트에서 해당 정보를 다시 SelectedList 로 보낸다. 
- 해당 정보를 랜더링

일단은 이렇게 생각해봤는데, 너무 불편한 것 같다. props 가 너무 많고, 만약 추후에 props 가 추가된다면 수많은 코드의 수정이 필요할 것이다. 

그래서 나는 다른 방법을 찾아봤고, `ContextAPI` 를 발견했다. 
# ContextAPI
ContextAPI 는 간단히 말해 컴포넌트 트리 안에서 데이터를 전역적으로 공유할 수 있게 해주는 방법이다. 이 방법을 사용하면 많은 수의 props 전달을 피할 수 있다. 

일단 데이터를 전역적으로 관리하는 `Provider` 컴포넌트를 만들어야 한다. 
{% highlight JSX%}
import React, { createContext, useContext, useState } from 'react';
const IngredientsContext = createContext();

export const IngredientsProvider = ({ children }) => {
   const [selectedIngredients, setSelectedIngredients] = useState([]);
   return (
      {% raw %}<IngredientsContext.Provider value={{ selectedIngredients, setSelectedIngredients }}>{% endraw %}
         {children}
      </IngredientsContext.Provider>
   );
};

export const useIngredients = () => useContext(IngredientsContext);
{% endhighlight %}
`createContext()` 를 통해 컨텍스트를 생성하고, `useIngredients()` 를 통해 컨텍스트를 사용한다. 

이후 App.js 에서 `Provider` 로 `IGDScroll` , `SelectedList` 를 감쌌다. 
{% highlight JSX%}
export default function App() {
   return (
      <div className='container mx-auto flex flex-col items-center justify-center gap-4 p-4 h-screen'>
         <h2 className='text-2xl mt-3'>메뉴 추천 - 갖고 있는 재료를 고르세요</h2>
         <IngredientsProvider>
            {% raw %}<ScrollArea className='topContainer flex gap-4 mb-4 mt-3' style={{ height: '40%' }}>{% endraw %}
               <div className='topContainer flex gap-4 mb-4 mt-3'>
                  <IGDScroll/>
                  <SelectedList/>
               </div>
            </ScrollArea>
            <LogButton/> 
            <FoodList/>
         </IngredientsProvider>
      </div>
   );
}
{% endhighlight %}

SelectedList 부터 살펴보자
{% highlight JSX%}
import React from "react";
import "../App.css";

import { ScrollArea } from "./ui/scroll-area"
import { Separator } from "./ui/separator"
import { useIngredients } from '../ingredientsContext';

function SelectedList() {
   const { selectedIngredients } = useIngredients();
   return (
      <>
      <h4 className="mb-4 pl-4 pt-4 text-sm font-medium leading-none">재료 선택</h4>
      {% raw %}<ScrollArea className="w-64 rounded-md pl-4 pr-4 pb-4" style={{height: "90%"}}>{% endraw %}
         <div className="p-1">
            {selectedIngredients.map((ingredient, index) => (
               <React.Fragment key={index}>
                  <div className="text-sm">
                     {ingredient}
                  </div>
                  <Separator className="my-2" />
               </React.Fragment>
            ))}
         </div>
      </ScrollArea>
      </>
   )
}
{% endhighlight %}
`useIngredients()` 를 사용해 컨텍스트를 사용한다(참고로 다른 컴포넌트에서 컨텍스트를 동시에 사용해도, 모두 같은 컨텍스트이기 때문에 동일한 상태를 유지한다.)

다음은 IGDScroll 이다. 코드가 길기 때문에 전체코드와 함께 중요한 부분만 따로 짚어보겠다. 
{% highlight JSX%}
import React from "react";
import "../App.css";
import { useState, useEffect } from "react";
import { ScrollArea } from "./ui/scroll-area"
import { Separator } from "./ui/separator"
import { useIngredients } from '../ingredientsContext';

function IGDScroll() {
   // 데이터와 검색 결과를 저장할 상태 변수 추가
   const [ingredients, setIngredients] = useState([]);
   const [searchResults, setSearchResults] = useState([]);
   const { selectedIngredients, setSelectedIngredients } = useIngredients();

   const addIngredient = (ingredient) => {
      if (!selectedIngredients.includes(ingredient)) {
         setSelectedIngredients([...selectedIngredients, ingredient]);
      }
   };

   //검색기능
   const handleSearch = (e) => {
      let keyword = e.target.value;
      if (keyword.trim() === '') { // 검색어가 없을 경우 모든 재료를 표시
         setSearchResults(ingredients);
      } else {
         const filteredData = ingredients.filter(data =>
   data.ingredientName.toLowerCase().includes(keyword.toLowerCase())
         );
         setSearchResults(filteredData); // 검색 결과 업데이트
      }
   };

   useEffect(() => {
   // 데이터를 가져오기
      const fetchData = async () => {
         try {
            const response = await fetch('http://localhost:8080/');
            if (!response.ok) {
               throw new Error('Data could not be fetched!');
            } else {
               const data = await response.json();
               setIngredients(data); // 상태 업데이트
               setSearchResults(data); // 초기 검색 결과 설정
            }
         } catch (error) {
            console.error("Error fetching data: ", error);
         }
      };
   
      fetchData(); 
   }, []); 

   return (
      <>
         <input type="text" onChange={handleSearch} placeholder="재료검색" className="{% raw %}css code ...{% endraw %}"/>
         {% raw %}<ScrollArea className="w-64 rounded-md pl-4 pr-4 pb-4" style={{height: "90%"}}>{% endraw %}
            <div className="p-1">
               {searchResults.map((ingredient, index) => (
                  <React.Fragment key={index}>
                     {% raw %}<div className="text-sm" onClick={() => addIngredient(ingredient.ingredientName)} style={{"cursor": "pointer"}}>{% endraw %}
                        {ingredient.ingredientName}
                     </div>
                     <Separator className="my-2" />
                  </React.Fragment>
               ))}
            </div>
         </ScrollArea>
      </>
   );
}

export default IGDList;
{% endhighlight %}
가장 먼저 봐야 할 것은 `SelectedList` 와 마찬가지로 컨텍스트를 사용하는 코드이다. 

{% highlight JSX%}
const { selectedIngredients, setSelectedIngredients } = useIngredients();
{% endhighlight %}

이후 재료 이름이 `onClick()` 이벤트에 의해 실행되면 `addIntgredient()` 가 실행된다.
{% highlight JSX%}
{% raw %}<div className="text-sm" onClick={() => addIngredient(ingredient.ingredientName)} style={{"cursor": "pointer"}}>{% endraw %}
   {ingredient.ingredientName}
</div>
{% endhighlight %}

선택된 재료를 통해 `selectedIngredients` 상태를 업데이트 한다. 
{% highlight JSX%}
const addIngredient = (ingredient) => {
   if (!selectedIngredients.includes(ingredient)) {
      setSelectedIngredients([...selectedIngredients, ingredient]);
   }
};
{% endhighlight %}

결과)
![](https://i.imgur.com/Nx3yabp.gif)

### 참고
- [다른 사람들이 안 알려주는 리액트에서 Context API 잘 쓰는 방법](https://velog.io/@velopert/react-context-tutorial)
