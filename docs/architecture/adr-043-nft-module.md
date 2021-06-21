# ADR 43: NFT Module

## Changelog

- 05.05.2021: Initial Draft

## Status

DRAFT

## Abstract

This ADR defines the `x/nft` module which as a generic implementation of the NFT API, roughly "compatible" with ERC721.

## Context

NFTs are more digital assets than only crypto arts, which is very helpful for accruing value to the Cosmos ecosystem. As a result, Cosmos Hub should implement NFT functions and enable a unified mechanism for storing and sending the ownership representative of NFTs as discussed in https://github.com/cosmos/cosmos-sdk/discussions/9065.

As was discussed in [#9065](https://github.com/cosmos/cosmos-sdk/discussions/9065), several potential solutions can be considered:

- irismod/nft and modules/incubator/nft
- CW721
- DID NFTs
- interNFT

Since NFTs functions/use cases are tightly connected with their logic, it is almost impossible to support all the NFTs' use cases in one Cosmos SDK module by defining and implementing different transaction types.

Considering generic usage and compatibility of interchain protocols including IBC and Gravity Bridge, it is preferred to have a generic NFT module design which handles the generic NFTs logic.

This design idea can enable composability that application-specific functions should be managed by other modules on Cosmos Hub or on other Zones by importing the NFT module.

The current design is based on the work done by [IRISnet team](https://github.com/irisnet/irismod/tree/master/modules/nft) and an older implementation in the [Cosmos repository](https://github.com/cosmos/modules/tree/master/incubator/nft).

## Decision

We will create a module `x/nft`, which contains the following functionality:

- Store and transfer NFTs utilizing `x/bank`.
- Mint and burn NFTs.
- Use `x/authz` to implement ERC721-style authorization.
- Query NFTs and their supply information.

### Types

#### Metadata

We define a `Metadata` model for `NFT Type`, which is comparable to an ERC721 contract on Ethereum, under which a collection of `NFT`s can be created and managed.

```protobuf
message Metadata {
  string type          = 1; // Required, unique key, alphanumeric
  string name          = 2;
  string symbol        = 3;
  string description   = 4;
  bool mint_restricted = 5;
  bool edit_restricted = 6;
}
```

- The `type` is the identifier of the NFT type/class.
- The `name` is a descriptive name of this NFT type.
- The `symbol` is the symbol usually shown on exchanges for this NFT type.
- The `description` is a detailed description of this NFT type.
- The `mint_restricted` flag, if set to true, indicates that only the issuer of this type can mint NFTs for it.
- The `edit_restricted` flag, if set to true, indicates that NFTs of this type cannot be edited once minted.

#### NFT

We define a general model for `NFT` as follows.

```protobuf
message NFT {
  string type              = 1; // The type of this NFT
  string id                = 2; // The identifier of this NFT
  string uri               = 3; 
  google.protobuf.Any data = 4;
}
```

The NFT conforms to the following specifications:

- The `id` is an immutable field used as a unique identifier within the scope of an NFT type. It is specified by the creator of the NFT and may be expanded to use DID in the future. NFT identifiers don't currently have a naming convention but can be used in association with existing Hub attributes, e.g., defining an NFT's identifier as an immutable Hub address allows its integration into existing Hub account management modules.
  We envision that identifiers can accommodate mint and transfer actions.
  The `id` is also the primary index for storing NFTs.

  ```
  {type}/{id} --> NFT (bytes)
  ```

- The `uri` points to an immutable off-chain resource containing more attributes about his NFT.

- The `data` is mutable field and allows attaching special information to the NFT, for example:

  - metadata such as the title of the work and URI.
  - immutable data and parameters (such actual NFT data, hash or seed for generators).
  - mutable data and parameters that change when transferring or when certain criteria are met (such as provenance).

  This ADR doesn't specify values that this field can take; however, best practices recommend upper-level NFT modules clearly specify their contents.
  Although the value of this field doesn't provide the additional context required to manage NFT records, which means that the field can technically be removed from the specification,
  the field's existence allows basic informational/UI functionality.

- The ownership of nft is controlled by the `x/bank` module and the `metadata` part will be converted to `banktypes.Metadata` is stored in the `x/bank` module.

### `Msg` Service

```protobuf
service Msg {
  rpc Issue(MsgIssue) returns (MsgIssueResponse);
  rpc Mint(MsgMint)   returns (MsgMintResponse);
  rpc Edit(MsgEdit)   returns (MsgEditResponse);
  rpc Send(MsgSend)   returns (MsgMsgSendResponse);
  rpc Burn(MsgBurn)   returns (MsgBurnResponse);
}

message MsgIssue {
  cosmos.nft.v1beta1.Metadata metadata = 1;
  string issuer                        = 2;
}
message MsgIssueResponse {}

message MsgMint {
  cosmos.nft.v1beta1.NFT nft = 1;
  string minter              = 2;
}
message MsgMintResponse {}

message MsgEdit {
  cosmos.nft.v1beta1.NFT nft = 1;
  string editor              = 2;
}
message MsgEditResponse {}

message MsgSend {
  string type     = 1;
  string id       = 2;
  string sender   = 3;
  string reveiver = 4;
}
message MsgSendResponse {}

message MsgBurn {
  string type      = 1;
  string id        = 2;
  string destroyer = 3;
}
message MsgBurnResponse {}
```

`MsgIssue` can be used to issue an NFT type/class, just like deploying an ERC721 contract on Ethereum.

`MsgMint` allows users to create new NFTs for a given type.

`MsgEdit` allows users to edit/update their NFTs.

`MsgSend` can be used to transfer the ownership of an NFT to another address.
**Note**: we could use `x/bank` to handle NFT transfer directly and do without this service. It's only for the sake of completeness of an ERC721 compatible API that we may choose to keep this service.

`MsgBurn` allows users to destroy their NFTs.

Other business logic implementations should be defined in other upper-level modules that import this NFT module. The implementation example of the server is as follows:

```go
type msgServer struct{
  k Keeper
}

func (m msgServer) Issue(ctx context.Context, msg *types.MsgIssue) (*types.MsgIssueResponse, error) {
  m.keeper.AssertTypeNotExist(msg.Metadata.Type)

  bz := m.keeper.cdc.MustMarshalBinaryBare(msg.Metadata)
  typeStore := m.keeper.getTypeStore(ctx)
  typeStore.Set(msg.Type, bz)
  
  bz := m.keeper.cdc.MustMarshalBinaryBare(msg.Issuer)
  typeOwnerStore := m.keeper.getTypeOwnerStore(ctx)
  typeOwnerStore.Set(msg.Type, bz)
  
  return &types.MsgIssueResponse{}, nil
}

func (m msgServer) Mint(ctx context.Context, msg *types.MsgMint) (*types.MsgMintResponse, error) {
  m.keeper.AssertTypeExist(msg.NFT.Type)
  m.keeper.AssertCanMint(msg.NFT.Type, msg.Minter)
  
  baseDenom := fmt.Sprintf("%s-%s", msg.NFT.Type, msg.NFT.Id)
  bkMetadata := bankTypes.Metadata{
    Symbol:      metadata.Symbol,
    Base:        baseDenom,
    Name:        metadata.Name,
    URI:         msg.NFT.URI,
    Description: metadata.Description,
  }
  
  m.keeper.bank.SetDenomMetaData(ctx, bkMetadata)
  mintedCoins := sdk.NewCoins(sdk.NewCoin(baseDenom, 1))
  m.keeper.bank.MintCoins(types.ModuleName, mintedCoins)
  m.keeper.bank.SendCoinsFromModuleToAccount(types.ModuleName, msg.Owner, mintedCoins)
  
  bz := m.keeper.cdc.MustMarshalBinaryBare(&msg.NFT)
  
  nftStoreByType := m.keeper.getNFTStoreByType(ctx, msg.NFT.Type)
  nftStoreByType.Set(msg.NFT.Id, bz)
  
  nftStore := m.keeper.getNFTStore(ctx)
  nftStore.Set(baseDenom, bz)
  
  return &types.MsgMintResponse{}, nil
}

func (m msgServer) Edit(ctx context.Context, msg *types.MsgEdit) (*types.MsgEditResponse, error) {
  m.keeper.AssertNFTExist(msg.Type, msg.Id)
  m.keeper.AssertCanEdit(msg.Type, msg.Id, msg.Editor)

  bz := m.keeper.cdc.MustMarshalBinaryBare(&msg.NFT)
  
  nftStoreByType := m.keeper.getNFTStoreByType(ctx, msg.NFT.Type)
  nftStoreByType.Set(msg.NFT.Id, bz)
  
  return &types.MsgEditResponse{}, nil
}

func (m msgServer) Send(ctx context.Context, msg *types.MsgSend) (*types.MsgSendResponse, error) {
  m.keeper.AssertNFTExist(msg.Type, msg.Id)

  sentCoins := sdk.NewCoins()  
  baseDenom := fmt.Sprintf("%s-%s", nft.Type, nft.Id)
  sentCoins = sentCoins.Add(sdk.NewCoin(baseDenom, 1)) 
  m.keeper.bank.SendCoins(ctx, msg.Sender, msg.Reveiver, sentCoins)
  
  return &types.MsgSendResponse{}, nil
}

func (m Keeper) Burn(ctx sdk.Context, msg *types.MsgBurn) (types.MsgBurnResponse,error) {
  m.keeper.AssertNFTExist(msg.Type, msg.Id)

  nftStoreByType := m.keeper.getNFTStoreByType(ctx, msg.Type)
  nft := nftStoreByType.Get(msg.Id)

  baseDenom := fmt.Sprintf("%s-%s", msg.Type, msg.Id)
  coins := sdk.NewCoins(sdk.NewCoin(baseDenom, 1))
  m.keeper.bank.SendCoinsFromAccountToModule(ctx, msg.Destroyer, types.ModuleName, coins)
  m.keeper.bank.BurnCoins(ctx, types.ModuleName, coins)

  // Delete bank.Metadata (keeper method not available)
  
  nftStoreByType := m.keeper.getNFTStoreByType(ctx, msg.NFT.Type)
  nftStoreByType.Delete(msg.Id)
  
  nftStore := m.keeper.getNFTStore(ctx)
  nftStore.Delete(baseDenom)

  return &types.MsgBurnResponse{}, nil
}
```

The upper application calls those methods by holding the MsgClient instance of the `x/nft` module. The execution authority of msg is guaranteed by the OCAPs mechanism.

The query service methods for the `x/nft` module are:

```proto
service Query {

  // NFT queries NFT details based on id.
  rpc NFT(QueryNFTRequest) returns (QueryNFTResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/nfts/{id}";
  }

  // NFTs queries all NFTs based on the optional owner.
  rpc NFTs(QueryNFTsRequest) returns (QueryNFTsResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/nfts";
  }

  // NFTsOf queries all NFTs based on the type.
  rpc NFTsOf(QueryNFTsOfRequest) returns (QueryNFTsOfResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/nfts/{type}";
  }

  // Supply queries the number of nft based on the type, same as totalSupply of ERC721
  rpc Supply(QuerySupplyRequest) returns (QuerySupplyResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/supply/{type}";
  }

  // Balance queries the number of NFTs based on the owner and type, same as balanceOf of ERC721
  rpc Balance(QueryBalanceRequest) returns (QueryBalanceResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/balance/{owner}/{type}";
  }

  // Type queries the definition of a given type
  rpc Type(QueryTypeRequest) returns (QueryTypeResponse) {
      option (google.api.http).get = "/cosmos/nft/v1beta1/types/{type}";
  }

  // Types queries all the types
  rpc Types(QueryTypesRequest) returns (QueryTypesResponse) {
      option (google.api.http).get = "/cosmos/nft/v1beta1/types";
  }
}

// QueryNFTRequest is the request type for the Query/NFT RPC method
message QueryNFTRequest {
  string id = 1;
}

// QueryNFTResponse is the response type for the Query/NFT RPC method
message QueryNFTResponse {
  cosmos.nft.v1beta1.NFT nft = 1;
}

// QueryNFTsRequest is the request type for the Query/NFTs RPC method
message QueryNFTsRequest {
  string                                 owner      = 1;
  cosmos.base.query.v1beta1.PageResponse pagination = 2;
}

// QueryNFTsResponse is the response type for the Query/NFTs RPC method
message QueryNFTsResponse {
  repeated cosmos.nft.v1beta1.NFT        nfts       = 1;
  cosmos.base.query.v1beta1.PageResponse pagination = 2;
}

// QueryNFTsOfRequest is the request type for the Query/NFTsOf RPC method
message QueryNFTsOfRequest {
  string                                 type       = 1;
  cosmos.base.query.v1beta1.PageResponse pagination = 2;
}

// QueryNFTsOfResponse is the response type for the Query/NFTsOf RPC method
message QueryNFTsOfResponse {
  repeated cosmos.nft.v1beta1.NFT        nfts       = 1;
  cosmos.base.query.v1beta1.PageResponse pagination = 2;
}

// QuerySupplyRequest is the request type for the Query/Supply RPC method
message QuerySupplyRequest{
  string type = 1;
}

// QuerySupplyResponse is the response type for the Query/Supply RPC method
message QuerySupplyResponse{
  uint64 amount = 1;
}

// QueryBalanceRequest is the request type for the Query/Balance RPC method
message QueryBalanceRequest{
  string owner = 1;
  string type  = 2;
}

// QueryBalanceResponse is the response type for the Query/Balance RPC method
message QueryBalanceResponse{
  uint64 amount = 1;
}

// QueryTypeRequest is the request type for the Query/Type RPC method
message QueryTypeRequest {
  string type = 1;
}

// QueryTypeResponse is the response type for the Query/Type RPC method
message QueryTypeResponse {
  cosmos.nft.v1beta1.Metadata metadata = 1;
}

// QueryTypesRequest is the request type for the Query/Types RPC method
message QueryTypesRequest {
  // pagination defines an optional pagination for the request.
  cosmos.base.query.v1beta1.PageRequest pagination = 1;
}

// QueryTypesResponse is the response type for the Query/Types RPC method
message QueryTypesResponse {
  repeated cosmos.nft.v1beta1.Metadata   metadatas = 1;
  cosmos.base.query.v1beta1.PageResponse pagination = 2;
}
```

## Consequences

### Backward Compatibility

No backward incompatibilities.

### Forward Compatibility

This specification conforms to the ERC-721 smart contract specification for NFT identifiers. Note that ERC-721 defines uniqueness based on (contract address, uint256 tokenId), and we conform to this implicitly because a single module is currently aimed to track NFT identifiers. Note: use of the (mutable) data field to determine uniqueness is not safe.s

### Positive

- NFT identifiers available on Cosmos Hub.
- Ability to build different NFT modules for the Cosmos Hub, e.g., ERC-721.
- NFT module which supports interoperability with IBC and other cross-chain infrastructures like Gravity Bridge

### Negative


### Neutral

- Other functions need more modules. For example, a custody module is needed for NFT trading function, a collectible module is needed for defining NFT properties.

## Further Discussions

For other kinds of applications on the Hub, more app-specific modules can be developed in the future:

- `x/nft/custody`: custody of NFTs to support trading functionality.
- `x/nft/marketplace`: selling and buying NFTs using sdk.Coins.

Other networks in the Cosmos ecosystem could design and implement their own NFT modules for specific NFT applications and use cases.

## References

- Initial discussion: https://github.com/cosmos/cosmos-sdk/discussions/9065
- x/nft: initialize module: https://github.com/cosmos/cosmos-sdk/pull/9174
- [ADR 033](https://github.com/cosmos/cosmos-sdk/blob/master/docs/architecture/adr-033-protobuf-inter-module-comm.md)