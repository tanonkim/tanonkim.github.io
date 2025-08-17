---
title: Design Patterns 4 : Provider Pattern
date: 2025-08-17 15:30:00 +0900
categories: [react, design-pattern, react-context]
tags: [react, design-pattern]
---

### 가정

앱 내의 여러 컴포넌트들이 데이터를 전달받아야 하는 상황이 있다.

**`props`** 를 통해서 데이터를 전달하는 방식이 있지만 앱 내의 모든 컴포넌트들이 데이터에 접근해야 하는 경우 이 작업을 하기 매우 번거로워진다.

그리고 아주 멀리있는 컴포넌트 계층까지 props를 내려주게 되면 props에 의존되는 컴포넌트는 나중에 리팩토링하기 어려워진다. : Props Drilling

```tsx
function App() {
  const data = { ... }

  return (
    <div>
      <SideBar data={data} />
      <Content data={data} />
    </div>
  )
}

const SideBar = ({ data }) => <List data={data} />
const List = ({ data }) => <ListItem data={data} />
const ListItem = ({ data }) => <span>{data.listItem}</span>

const Content = ({ data }) => (
  <div>
    <Header data={data} />
    <Block data={data} />
  </div>
)
const Header = ({ data }) => <div>{data.title}</div>
const Block = ({ data }) => <Text data={data} />
const Text = ({ data }) => <h1>{data.text}</h1>
```

`App` 컴포넌트가 있고. 특정 데이터를 가지고 있다고 가정해 보자.

컴포넌트 트리의 마지막 노드에는 `ListItem`, `Header`, `Text` 컴포넌트가 있고 `App`이 가진 데이터를 필요로 한다. 이 컴포넌트들에게 데이터를 주려면 여러 중간 컴포넌트들에게 데이터를 내려주어야 한다. ⇒ Props Drilling

해당 방식으로 props를 내리는 것은 지저분하다. data라는 프로퍼티 이름을 변경하는 경우 모든 컴포넌트를 수정해야 한다. → 앱이 커지면 수정이 점점 더 어려워진다.

### Provider 패턴

이 경우 Provider패턴은 매우 유용하다. 각 레이어에 직접 데이터를 주지 않고도 여러 컴포넌트에게 데이터에 접근할 수 있게 구현할 수 있다.

1. 먼저 모든 컴포넌트를 Provider로 감싼다.

   Provider는 HOC로 ‘Context’객체를 제공한다. React가 제공하는 createContext 메서드를 활용해서 Context 객체를 만들어 낼 수 있다.

   ```tsx

   const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

   export const **ThemeProvider**: React.FC<{ children: React.ReactNode }> = ({
       children,
   }: {
       children: React.ReactNode;
   }) => {
       const [theme, setTheme] = useState(() => {
           try {
               const stored = (localStorage.getItem('theme') as Theme) || null;
               return stored ?? 'light';
           } catch {
               return 'light';
           }
       });

       const toggle = () => {
           setTheme((t) => {
               const next = t === 'light' ? 'dark' : 'light';
               localStorage.setItem('theme', next);

               return next;
           });
       };

       const value = useMemo(
           () => ({
               theme,
               setTheme,
               toggle,
           }),
           [theme]
       );
       return <ThemeContext.Provider value={value}>
   					    {children}
   						</ThemeContext.Provider>;
   };

   // App.tsx
   const root = createRoot(document.getElementById('root')!);

   root.render(
       <ThemeProvider>
           <AuthProvider>
               <App /> // value를 어떻게 받아야 하나?
           </AuthProvider>
       </ThemeProvider>
   );

   ```

   Provider 컴포넌트는 value라는 이름으로 prop으로 하위컴포넌트들에 내려줄 데이터를 받는다.

   이제 각 컴포넌트에게 value를 일일히 넘기지 않아도 된다. 그러면 <App /> 컴포넌트에서 value를 어떻게 접근할까?

   각 컴포넌트는 useContext 훅을 활용하여 value에 접근할 수 있다.

   ```tsx
   export default function App() {
     const { theme } = useTheme();

     const bg = theme === "light" ? "var(--bg-light)" : "var(--bg-dark)";
     const color = theme === "light" ? "var(--text-light)" : "var(--text-dark)";

     return (
       <div className="app" style={{ backgroundColor: bg, color }}>
         <h1>React Context 예제 (TypeScript)</h1>
         <ThemeToggle />
         <LoginForm />
         <Profile />
         <ImageList />
         <section className="card">
           <h2>설명</h2>
           <p>
             이 예제는 ThemeContext와 AuthContext를 보여주며, 상태(state)와
             액션(dispatch)을 분리하고, useMemo/useCallback으로 value/함수
             참조를 안정화하는 패턴을 사용합니다.
           </p>
         </section>
       </div>
     );
   }

   // ThemeContext.tsx
   interface ThemeContextType {
     theme: Theme;
     setTheme: (t: Theme) => void;
     toggle: () => void;
   }

   export const useTheme = (): ThemeContextType => {
     const ctx = useContext(ThemeContext);
     if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
     return ctx;
   };
   ```

   App 컴포넌트는 useContext 훅이 존재하는 useTheme 커스텀 훅을 통해서 value에 접근할 수 있다. 이 커스텀 훅은 value와 연관된 `ThemeContext`를 받아 value를 읽고 쓸 수 있는 컨텍스트를 제공한다.

   이로써 value를 필요하지 않는 컴포넌트는 value를 prop으로 받지 않게 되었다. prop drilling도 필요 없어졌으며 전보다 훨씬 리펙토링하기도 수월해졌다.

   참고 : useTheme 훅으로 consumer가 Provider 밖에서 사용되면 바로 에러로 알려준다. (실수 방지)

   ### 장점

   - 컴포넌트 트리의 각 노드에 데이터를 전달하지 않아도 다수의 컴포넌트에 데이터를 전달할 수 있다.
   - 리펙토링 과정에 개발자가 실수할 확률을 줄여준다.
     이전에는 prop의 이름을 변경하기 위해서 모든 컴포넌트를 찾아다니며 코드를 수정해야 했다.
   - prop-drilling을 하지 않아도 된다.
     Provider 패턴을 이용하면 데이터가 필요없는 컴포넌트에 불필요하게 prop을 받을 필요가 없어진다.

   ### 단점

   - Provider 패턴을 과하게 사용할 경우 특정 상황에서 성능 이슈가 발생할 수 있다.
     컨텍스트를 참조하는 모든 컴포넌트는 컨텍스트 변경시마다 모두 리렌더링된다.
     → useMemo로 value를 고정해 불필요한 재렌더링을 줄인다.
