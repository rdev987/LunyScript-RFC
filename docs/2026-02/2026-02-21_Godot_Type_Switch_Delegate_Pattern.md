# 2026-02-21_Godot_Type_Switch_Delegate_Pattern

Godot: centralized proxy strategy for once-only type-checking ...

Goal: use `node.transform.position` across entire API without worrying about whether the underlying object is Node2D, Node3D, or logical (Node).

```
using Godot;
using System;

public class NodeProxy {    // <== GodotNode (LunyObject subclass)
private readonly Node _node;
public TransformProxy transform { get; }    // <== LunyTransform + NodeTransform

    public NodeProxy(Node node) {
        _node = node;
        transform = new TransformProxy(node);
    }

    public class TransformProxy {
        // Centralized delegates for get/set logic
        private readonly Func<Vector3> _getPos;
        private readonly Action<Vector3> _setPos;

        public TransformProxy(Node node) {
            // TYPE CHECKING HAPPENS ONLY ONCE HERE
            if (node is Node3D n3) {
                _getPos = () => n3.Position;            // make lambdas static, alloc only once?
                _setPos = (v) => n3.Position = v;
            }
            else if (node is Node2D n2) {
                _getPos = () => new Vector3(n2.Position.X, n2.Position.Y, 0);
                _setPos = (v) => n2.Position = new Vector2(v.X, v.Y);
            }
            else {
                // Default "Safe" behavior for base Nodes
                _getPos = () => Vector3.Zero;
                _setPos = (v) => { }; 
            }
        }

        // Clean properties with zero logic/type-checks
        public Vector3 position {
            get => _getPos();
            set => _setPos(value);
        }
    }
}

// Static lambda pattern

public class NodeProxy {
    // 1. Static Accessor Definitions (Shared by all instances)
    private static readonly Func<Node, Vector3> Get3D = (n) => ((Node3D)n).Position;
    private static readonly Action<Node, Vector3> Set3D = (n, v) => ((Node3D)n).Position = v;

    private static readonly Func<Node, Vector3> Get2D = (n) => {
        var p = ((Node2D)n).Position;
        return new Vector3(p.X, p.Y, 0);
    };
    private static readonly Action<Node, Vector3> Set2D = (n, v) => ((Node2D)n).Position = new Vector2(v.X, v.Y);

    private static readonly Func<Node, Vector3> GetNone = (n) => Vector3.Zero;
    private static readonly Action<Node, Vector3> SetNone = (n, v) => { };

    private readonly Node _node;
    private readonly Func<Node, Vector3> _getter;
    private readonly Action<Node, Vector3> _setter;

    public NodeProxy(Node node) {
        _node = node;
        // 2. Assign the shared static reference ONCE
        if (node is Node3D) { _getter = Get3D; _setter = Set3D; }
        else if (node is Node2D) { _getter = Get2D; _setter = Set2D; }
        else { _getter = GetNone; _setter = SetNone; }
    }

    public Vector3 Position {
        get => _getter(_node); // No new allocation here
        set => _setter(_node, value);
    }
}
```
