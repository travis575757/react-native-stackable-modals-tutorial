# Stackable Modal Demo

Define a modal context which manages the modal instances

```javascript
const StackableModalContext = createContext({
  mountModal: () => {},
});

/**
 * Provider for modals which can be rendered simultaneously (stacked)
 */
function StackableModalProvider({ children }) {
  const keysRef = useRef([]);
  const stateRef = useRef([]);
  const onShowListRef = useRef([]);
  const [state, setState] = useState([]);

  useEffect(() => {
    if (state.length > 0) {
      // invoke the onShow callback if it exists for this view
      onShowListRef.current.forEach(callback => {
        callback();
      });
      onShowListRef.current = [];
    }
  }, [state]);

  const mountModal = useCallback((modalChildren) => {
    // compute a random unique identifier
    const timestamp = Date.now();
    const randomNum = Math.random();
    const id = `${timestamp.toString(36)}-${randomNum.toString(36).substr(2, 9)}`;
    // push the modal data
    stateRef.current.push(modalChildren);
    keysRef.current.push(id);
    setState([...stateRef.current]);

    return [
      () => {
        const idx = keysRef.current.indexOf(id);
        keysRef.current.splice(idx, 1);
        stateRef.current.splice(idx, 1);
        setState([...stateRef.current]);
      },
      (updatedChildren) => {
        const idx = keysRef.current.indexOf(id);
        stateRef.current[idx] = updatedChildren;
        setState([...stateRef.current]);
      },
    ];
  }, []);

  return (
    <StackableModalContext.Provider value={{ mountModal }}>
      {children}
      {state.map((item, index) => (
        <React.Fragment key={index}>{item}</React.Fragment>
      ))}
    </StackableModalContext.Provider>
  );
}
```

Define a new base modal element which uses this context. This is the component that acts like a normal react Modal component

```javascript
/**
 * Component to use as a parent for all modals
 */
function StackableModal({ children, visible }) {
  const { mountModal } = useContext(StackableModalContext);
  const updateRef = useRef(null);

  useEffect(() => {
    if (visible) {
      const handler = BackHandler.addEventListener('hardwareBackPress', () => true);
      return () => handler.remove();
    }
  }, [visible]);

  function getView(children) {
    return (
      <View
        pointerEvents={visible ? 'auto' : 'none'}
        style={{ position: 'absolute', width: '100%', height: '100%', opacity: visible ? 1 : 0 }}
      >
        {children}
      </View>
    );
  }

  useEffect(() => {
    const [unmount, update] = mountModal(getView(children));
    updateRef.current = update;
    return unmount;
  }, []);

  useEffect(() => {
    if (updateRef.current) {
      updateRef.current(getView(children));
    }
  }, [children]);

  return null;
}
```

Now you can basically use it like a normal react modal (obviously it doesn't have all the same props and stuff but that can be added as you need)

```javascript
function MyCustomModal(props) {
  return (
    <StackableModal visible={props.visible}>
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center', backgroundColor: 'rgba(0,0,0,0.5)' }}>
        <View style={{ backgroundColor: 'white', padding: 20, borderRadius: 10, width: '80%', alignItems: 'center' }}>
          <Text style={{ marginBottom: 20 }}>Hello, this is a modal!</Text>
          <Button title="Close Modal" onPress={() => {
            props.setVisible(false);
          }} />
        </View>
      </View>
    </StackableModal>
  );
}

export default function App() {
  const [visible1, setVisible1] = useState(false);
  const [visible2, setVisible2] = useState(false);
  const [visible3, setVisible3] = useState(false);
  return (
    <StackableModalProvider>
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <Text>Main Application View</Text>
        <MyCustomModal title="modal 1" visible={visible1} setVisible={setVisible1} />
        <MyCustomModal title="modal 2" visible={visible2} setVisible={setVisible2} />
        <MyCustomModal title="modal 3" visible={visible3} setVisible={setVisible3} />
        <Button title='show' onPress={() => {
          setVisible1(true)
          setVisible2(true)
          setVisible3(true)
        }} />
      </View>
    </StackableModalProvider>
  );
}
```

Demo Here: https://snack.expo.dev/@travis575757/993dec
