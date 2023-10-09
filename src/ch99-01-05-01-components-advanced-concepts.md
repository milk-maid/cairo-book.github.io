# Advanced components concepts

## Components under the hood

The purpose of this section is to explain what exactly is going on in the
compiler in the following lines:

```rust
{{#include ../listings/ch99-starknet-smart-contracts/components/listing_01_ownable/src/component.cairo:impl_signature}}
```

Before we zoom into the Upgradable impl, we need to discuss embeddable impls, a
new feature introduced in Cairo v2.3.0. An impl of a starknet interface trait
(that is, a trait annotated with the #[starknet::interface] attribute) can be
embeddable. One can embed embeddable impls in any contract, consequently adding
new entry points and changing the ABI. Let’s consider the following example
(which is not using components):

```rust
{{#include ../listings/ch99-starknet-smart-contracts/components/no_listing_01_embeddable/src/lib.cairo}}
```

<!-- TODO link impl alias section -->

By embedding the impl with the following impl alias we have added MySimpleImpl
and SimpleTrait to the ABI of the contract, and can call ret4 externally.

Now that we’re more familiar with the embedding mechanism, we can go back to the
Upgradable impl inside our component:

Zooming in, we can notice two component-specific changes:

Upgradable is dependent on an implementation of the HasComponent<TContractState>
trait

This dependency allows the compiler to generate an impl that can be used on
every contract. That is, the compiler will generate an impl that wraps any
function in Upgradeable, replacing the self: ComponentState<TContractState>
argument with self: TContractState, where access to the component state is made
via get_component function in the HasComponent<TContractState> trait.

To get a more complete picture of what’s being generated behind the scenes, the
following trait is being generated by the compiler per a component module:

```rust
// generated per component
trait HasComponent<TContractState> {
    fn get_component(self: @TContractState) -> @ComponentState<TContractState>;
    fn get_component_mut(ref self: TContractState) -> ComponentState<TContractState>;
    fn get_contract(self: @ComponentState<TContractState>) -> @TContractState;
    fn get_contract_mut(ref self: ComponentState<TContractState>) -> TContractState;
    fn emit<S, impl IntoImp: traits::Into<S, Event>>(ref self: ComponentState<TContractState>, event: S);
}
```

In our context ComponentState<TContractState> is a type specific to the
upgradable component, i.e. it has members based on the storage variables defined
in upgradable_component::Storage. Moving from the generic TContractState to
ComponentState<TContractState> will allow us to embed Upgradable in any contract
that wants to use it. The opposite direction (ComponentState<TContractState> to
ContractState) is useful for dependencies (see the Upgradable component
depending on an OwnableTrait implementation example)

To put it briefly, one should think of an implementation of the above
HasComponent<T> as saying: “Contract whose state is T has the upgradable
component”.

Upgradable is annotated with the embeddable_as(<name>) attribute:

embeddable_as is similar to embeddable; it only applies to impls of starknet
interface traits and allows embedding this impl in a contract module. That
said,embeddable_as(<name>) has another role in the context of components.
Eventually, when embedding Upgradable in some contract, we expect to get an impl
with the following function:

`fn upgrade(ref self: ContractState, new_class_hash: ClassHash)`

Note that while starting with a function receiving the generic type
ComponentState<TContractState>, we want to end up with a function receiving
ContractState. This is where embeddable_as(<name>) comes in. To see the full
picture, we need to see what is the impl generated by the compiler due to the
embeddable_as(UpgradableImpl) annotation:

```rust
// generated
#[starknet::embeddable]
impl UpgradableImpl<TContractState, +HasComponent<TContractState>> of UpgradableTrait<TContractState> {
    fn upgrade(ref self: TContractState, new_class_hash: ClassHash) {
        let mut component = self.get_component_mut();
        component.upgrade(new_class_hash, )
    }
}
```

Note that thanks to having an impl of HasComponent<TContractState>, the compiler
was able to wrap our upgrade function in a new impl that doesn’t directly know
about the ComponentState type. UpgradableImpl, whose name we chose when writing
embeddable_as(UpgradableImpl), is the impl that we will embed in a contract that
wants upgradability.

To complete the picture, we look at the following lines inside
counter_contract.cairo

```rust
#[abi(embed_v0)]
impl Upgradable = upgradable_component::UpgradableImpl<ContractSate>;
```

We’ve seen how UpgradableImpl was generated by the compiler inside
upgradable.cairo. The above lines use the Cairo v2.3.0 impl embedding mechanism
alongside the impl alias syntax. We’re instantiating the generic
UpgradableImpl<TContractState> with the concrete type ContractState. Recall that
UpgradableImpl<TContractState> has the HasComponent<TContractState> generic impl
param. An implementation of this trait is generated by the component! macro.
Note that only the using contract could have implemented this trait since only
it knows about both the contract state and the component state.

<!-- TODO -->

# Troubleshooting

You might encounter some errors when trying to implement components.
Unfortunately, some of them lack meaningful error messages to help debug. This
section aims to provide you with some pointers to help you debug your code.

- `Trait not found. Not a trait.`

  This error can occur when you're not importing the component's impl block
  correctly in your contract. Make sure to respect the following syntax:

  ```rust
  #[abi(embed_v0)]
  impl IMPL_NAME = upgradable::EMBEDDED_NAME<ContractState>
  ```

  Referring to our previous example, this would be:

  ```rust
  #[abi(embed_v0)]
  impl OwnableImpl = upgradable::Ownable<ContractState>
  ```

- `Plugin diagnostic: name is not a substorage member in the contract's Storage.
Consider adding to Storage: (...)`

  The compiler helps you a lot debugging this by giving you recommendation on
  the action to take. Basically, you forgot to add the component's storage to
  your contract's storage. Make sure to add the path to the component's storage
  annotated with the `#[substorage(v0)]` attribute to your contract's storage.

- `Plugin diagnostic: name is not a nested event in the contract's Event enum.
Consider adding to the Event enum:`

  Similar to the previous error, the compiler, you forgot to add the component's
  events to your contract's events. Make sure to add the path to the component's
  events to your contract's events.

- Components functions are not accessible externally

  This can happen if you forgot to annotate the component's impl block with
  `#[abi(embed_v0)]`. Make sure to add this annotation when embedding the
  component's impl in your contract.