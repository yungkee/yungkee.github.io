---
layout: post
title: Replication Graph – How To Reduce Network Bandwidth In Unreal
modified:
categories: 
excerpt: UnrealEngine
tags: [Notes]
image:
  feature: note.jpg
  thumb: RepGragh1.jpg
date: 2024-02-17T02:54:48+05:30
---

## Replication Graph – How To Reduce Network Bandwidth In Unreal

A crucial factor in developing networked games is managing the required bandwidth to ensure a stable and smooth gameplay experience. Sending or receiving excessive data can overload players with slower internet connections, negatively impacting the game experience. To avoid this, certain data must be prioritized, while others are delayed. This becomes especially challenging in games like Fortnite, which hosts 100 players per map, prompting Epic Games to introduce the Replication Graph as a solution.

Maintaining bandwidth limits is often manageable when only a few players are involved in a session. However, as the number of players increases, the amount of data required to synchronize everyone also grows. This effect is exponential, as it's not just about sending more data, but also about delivering it to a larger number of players.

One of Unreal’s key systems for addressing this issue is [Actor Relevancy](https://dev.epicgames.com/documentation/en-us/unreal-engine/actor-relevancy-and-priority?application_version=4.27). This system reduces bandwidth usage by removing actors from a player’s view when they are too far away. As a result, no updates are needed for these actors, which eliminates the need to transmit unnecessary data.

![Replication Graph3](videoframe_2115.png)

However, distance-based culling alone isn't enough. A common gameplay mechanic, like fog of war, requires players to only see enemy team members if their teammates have line of sight. [The Replication Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/replication-graph?application_version=4.27) can be used here to establish more advanced rules about what data each player should receive. In fog of war scenarios, this feature can further restrict access to sensitive information, like enemy positions, thus preventing cheating (e.g., wall hacks). Moreover, this optimization not only reduces network traffic but also improves CPU performance by freeing up server resources.

## Setting Up The Replication Graph

First of all we need to enable the plugin like any other plugin, by adding “ReplicationGraph” to PrivateDependencyModuleNames in your build.cs and adding the plugin to enabled plugins in your .uproject. 

```cs
    "Plugins": [
    {
      "Name": "ReplicationGraph",
      "Enabled": true
    }
```
We can then go ahead to create a new class that inherits from UReplicationGraph.

```cs
  
#pragma once

#include "CoreMinimal.h"
#include "ReplicationGraph.h"
#include "TutorialRepGraph.generated.h"

UCLASS(Blueprintable)
class REPGRAPH_API UTutorialRepGraph : public UReplicationGraph
{
  GENERATED_BODY()
};

```

Finally we need to add a line to our DefaultEngine.ini which will tell the NetDriver which replication graph to use. I’m going to create a blueprint child of our new UTutorialRepGraph class to make it a bit easier to set up any references or settings so my DefaultEngine.ini will point to that. There is also an option to bind to a delegate if you require different rep graphs for different game modes which you can read more about in Unreal’s documentation.

```cs
  
[/Script/OnlineSubsystemUtils.IpNetDriver]
ReplicationDriverClassName="/Game/BP_TutorialRepGraph.BP_TutorialRepGraph_C"

```

## Setting Up The Connection Manager And Nodes

A node is how we specify rules for who should receive what data and we can add or remove actors to these nodes as we need. Each connection is assigned their own connection manager which holds the nodes for any unique rules. For this example I’d like to recreate a fog of war scenario where players can always see other members of their own team, but can only see the opposing players when they have entered the player’s line of sight. To visualise this I have set up characters which get assigned to a random team when spawned and coloured to match.

![Replication Graph1](rep1.jpg)


The first step is to set up the different type of nodes we will need to make this happen. We have two functions we must override to do this: InitGlobalGraphNodes and InitConnectionGraphNodes. These functions give us the opportunity to set up nodes that all connections and actors will require and nodes that specify rules per player.

```cs
  
void UTutorialRepGraph::InitGlobalGraphNodes()
{
  Super::InitGlobalGraphNodes();

  // Create the always relevant node
  AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_AlwaysRelevant_WithPending>();
  AddGlobalGraphNode(AlwaysRelevantNode);
}

void UTutorialRepGraph::InitConnectionGraphNodes(UNetReplicationGraphConnection* ConnectionManager)
{
  Super::InitConnectionGraphNodes(ConnectionManager);
  
  // Create the connection graph for the incoming connection
  UTutorialConnectionManager* TutorialConnectionManager = Cast<UTutorialConnectionManager>(ConnectionManager);

  if (ensure(TutorialRepGraph))
  {
    TutorialConnectionManager->AlwaysRelevantForConnectionNode = CreateNewNode<UReplicationGraphNode_AlwaysRelevant_ForConnection>();
    AddConnectionGraphNode(TutorialRepGraph->AlwaysRelevantForConnectionNode, ConnectionManager);
    
    TutorialConnectionManager->TeamConnectionNode = CreateNewNode<UReplicationGraphNode_AlwaysRelevant_ForTeam>();
    AddConnectionGraphNode(TutorialRepGraph->TeamConnectionNode, ConnectionManager);
  }
}

```

We can add all actors to the AlwaysRelevantNode that every player needs access to, such as the game state. There is also a bool here, bRequiresPrepareForReplicationCall which will enable calls to PrepareForReplication before a connection is ready to replicate. We can talk more about that later.

```cs
  
UReplicationGraphNode_AlwaysRelevant_WithPending::UReplicationGraphNode_AlwaysRelevant_WithPending()
{
  // Call PrepareForReplication before replication once per frame
  bRequiresPrepareForReplicationCall = true;
}

void UReplicationGraphNode_AlwaysRelevant_WithPending::PrepareForReplication()
{
  UTutorialRepGraph* ReplicationGraph = Cast<UTutorialRepGraph>(GetOuter());
  ReplicationGraph->HandlePendingActorsAndTeamRequests();
}
```

Each connection has its own connection manager in which we specify the class in the constructor and it is also added to its own UReplicationGraphNode_AlwaysRelevant_ForConnection and UReplicationGraphNode_AlwaysRelevant_ForTeam nodes. The _ForConnection node is used for actors which are only relevant to this connection like the PlayerController. The _ForTeam node is what we will use to show other members of the same team. We’re using GatherActorListsForConnection here which will be called every tick to provide a list of nodes with the same team index as itself and will therefore be relevant.

```cs
  
UTutorialRepGraph::UTutorialRepGraph()
{
  // Specify the connection graph class to use
  ReplicationConnectionManagerClass = UTutorialConnectionManager::StaticClass();
}

UCLASS()
class UReplicationGraphNode_AlwaysRelevant_ForTeam : public UReplicationGraphNode_ActorList
{
  GENERATED_BODY()

  virtual void GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params) override;
  virtual void GatherActorListsForConnectionDefault(const FConnectionGatherActorListParameters& Params);
};

UCLASS()
class UTutorialConnectionManager : public UNetReplicationGraphConnection
{
  GENERATED_BODY()

public:
  // UReplicationGraphNode_AlwaysRelevant_ForConnection is a node type provided by Epic for actors always relevant to a connection
  UPROPERTY()
  UReplicationGraphNode_AlwaysRelevant_ForConnection* AlwaysRelevantForConnectionNode;
  
  UPROPERTY()
  UReplicationGraphNode_AlwaysRelevant_ForTeam* TeamConnectionNode;

  int32 Team = -1;
};

void UReplicationGraphNode_AlwaysRelevant_ForTeam::GatherActorListsForConnection(
  const FConnectionGatherActorListParameters& Params)
{
  // Get all other team members with the same team ID from ReplicationGraph->TeamConnectionListMap
  UTutorialRepGraph* ReplicationGraph = Cast<UTutorialRepGraph>(GetOuter());
  const UTutorialConnectionManager* ConnectionManager = Cast<UTutorialConnectionManager>(&Params.ConnectionManager);
  if (ReplicationGraph && ConnectionManager && ConnectionManager->Team != -1)
  {
    if (TArray<UTutorialConnectionManager*>* TeamConnections = ReplicationGraph->TeamConnectionListMap.GetConnectionArrayForTeam(ConnectionManager->Team))
    {
      for (const UTutorialConnectionManager* TeamMember : *TeamConnections)
      {
        TeamMember->TeamConnectionNode->GatherActorListsForConnectionDefault(Params);
      }
    }
  }
  else
  {
    Super::GatherActorListsForConnection(Params);
  }
}

void UReplicationGraphNode_AlwaysRelevant_ForTeam::GatherActorListsForConnectionDefault(
  const FConnectionGatherActorListParameters& Params)
{
  Super::GatherActorListsForConnection(Params);
}
```

## Adding Each Actor To The Correct Node

Now that we’re set up and each connection gets its own connection manager we need to add each actor to the correct node. Every actor will pass through the RouteAddNetworkActorToNodes function where we can then call NotifyAddNetworkActor if that actor needs a specific rule. Below is a simple example of what you can do to get things working but you’ll likely have many actors with different rules and would benefit from creating a better system for this (as shown in the [Locus Replication Graph ](https://github.com/locus84/LocusReplicationGraph) example).

```cs

void UTutorialRepGraph::RouteAddNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo,
                                                    FGlobalActorReplicationInfo& GlobalInfo)
{
  // All clients must receive game states and player states
  if (ActorInfo.Class->IsChildOf(AGameStateBase::StaticClass()) || ActorInfo.Class->IsChildOf(APlayerState::StaticClass()))
  {
    AlwaysRelevantNode->NotifyAddNetworkActor(ActorInfo);
  }
  // If not we see if it belongs to a connection
  else if (UTutorialConnectionManager* ConnectionManager = GetTutorialConnectionManagerFromActor(ActorInfo.GetActor()))
  {
    if (ActorInfo.Actor->bOnlyRelevantToOwner)
    {
      ConnectionManager->AlwaysRelevantForConnectionNode->NotifyAddNetworkActor(ActorInfo);
    }
    else
    {
      ConnectionManager->TeamConnectionNode->NotifyAddNetworkActor(ActorInfo);
    }
  }
  else if(ActorInfo.Actor->GetNetOwner())
  {
    // Add to PendingConnectionActors if the net connection is not ready yet
    PendingConnectionActors.Add(ActorInfo.GetActor());
  }
}
```
GetTutorialConnectionManagerFromActor is a simple function that will either get or create a connection manager for any actor with a net connection.

```cs

UTutorialConnectionManager* UTutorialRepGraph::GetTutorialConnectionManagerFromActor(const AActor* Actor)
{
  if (Actor)
  {
    if (UNetConnection* NetConnection = Actor->GetNetConnection())
    {
      if (UTutorialConnectionManager* ConnectionManager = Cast<UTutorialConnectionManager>(FindOrAddConnectionManager(NetConnection)))
      {
        return ConnectionManager;
      }
    }
  }
  
  return nullptr;
}
```

Unfortunately there is some time where the net connection is not available yet so this function will return a nullptr. You can see above in RouteAddNetworkActorToNodes we’re catching that case if the actor has a net owner and adding it to the PendingConnectionActors array so that we can process it later.

```cs

void UTutorialRepGraph::HandlePendingActorsAndTeamRequests()
{ 
  // Set up all pending connections
  if (PendingConnectionActors.Num() > 0)
  {
    TArray<AActor*> PendingActors = MoveTemp(PendingConnectionActors);

    for (AActor* Actor : PendingActors)
    {
      if (IsValid(Actor))
      {
        FGlobalActorReplicationInfo& GlobalInfo = GlobalActorReplicationInfoMap.Get(Actor);
        RouteAddNetworkActorToNodes(FNewReplicatedActorInfo(Actor), GlobalInfo);
      }
    }
  }
}
```

This function is called by the previous PrepareForReplication call and will just repeat any setup logic for each of the PendingConnectionActors to make sure they’re properly set up and assigned to the correct nodes.

This is the bulk of the player setup but it does leave out one critical piece of the puzzle which is setting the team index in the connection graph.

## Setting The Player’s Team


```cs

void UTutorialRepGraph::SetTeamForPlayerController(APlayerController* PlayerController, int32 Team)
{
  if (PlayerController)
  {
    if (UTutorialConnectionManager* ConnectionManager = GetTutorialConnectionManagerFromActor(PlayerController))
    {
      const int32 CurrentTeam = ConnectionManager->Team;
      if (CurrentTeam != Team)
      {
        // Remove the connection to the old team list
        if (CurrentTeam != -1)
        {
          TeamConnectionListMap.RemoveConnectionFromTeam(CurrentTeam, ConnectionManager);
        }

        // Add the graph to the new team list
        if (Team != -1)
        {
          TeamConnectionListMap.AddConnectionToTeam(Team, ConnectionManager);
        }
        
        ConnectionManager->Team = Team;
      }
    }
    else
    {
      // Add to PendingTeamRequests if the net connection is not ready yet
      PendingTeamRequests.Emplace(Team, PlayerController);
    }
  }
}
```
SetTeamForPlayerController is a function that adds the PlayerController to the correct team index in the TeamConnectionListMap. The TeamConnectionListMap is just a map with the team index as the key and a list of each connection graph assigned to that team as the value, so that we can quickly query it as needed (and we’ve already used it in GatherActorListsForConnection). You can see here we may have the same issue as earlier where we try to set the team before the net connection is available, so we also have to store an array of pending team requests and also add that setup functionality to HandlePendingActorsAndTeamRequests.


```cs

void UTutorialRepGraph::HandlePendingActorsAndTeamRequests()
{
  // Setup all pending team requests
  if(PendingTeamRequests.Num() > 0)
  {
    TArray<TTuple<int32, APlayerController*>> TempRequests = MoveTemp(PendingTeamRequests);

    for (const TTuple<int32, APlayerController*>& Request : TempRequests)
    {
      if (IsValid(Request.Value))
      {
        SetTeamForPlayerController(Request.Value, Request.Key);
      }
    }
  }
  
  //... Handle Pending Actors
}
```

The FTeamConnectionListMap has a few functions to improve the usability of adding and removing connection graphs from the map.


```cs

struct FTeamConnectionListMap : TMap<int32, TArray<UTutorialConnectionManager*>>
{
  TArray<UTutorialConnectionManager*>* GetConnectionArrayForTeam(int32 Team);
  
  void AddConnectionToTeam(int32 Team, UTutorialConnectionManager* ConnManager);
  void RemoveConnectionFromTeam(int32 Team, UTutorialConnectionManager* ConnManager);
};

TArray<UTutorialConnectionManager*>* FTeamConnectionListMap::GetConnectionArrayForTeam(int32 Team)
{
  return Find(Team);
}

void FTeamConnectionListMap::AddConnectionToTeam(int32 Team, UTutorialConnectionManager* ConnManager)
{
  TArray<UTutorialConnectionManager*>& TeamList = FindOrAdd(Team);
  TeamList.Add(ConnManager);
}

void FTeamConnectionListMap::RemoveConnectionFromTeam(int32 Team, UTutorialConnectionManager* ConnManager)
{
  if (TArray<UTutorialConnectionManager*>* TeamList = Find(Team))
  {
    TeamList->RemoveSwap(ConnManager);
    
    // Remove the team from the map if there are no more connections
    if (TeamList->Num() == 0)
    {
      Remove(Team);
    }
  }
}
```

And finally, outside of the connection graph we need to call into SetTeamForPlayerController whenever the team index changes. As this is a simple example we just do it once in the BeginPlay of the PlayerState.

```cs

void ATutorialPlayerState::BeginPlay()
{
  Super::BeginPlay();

  if (HasAuthority())
  {
    Team = FMath::RandBool() ? 0 : 1;
    
    if (const UWorld* World = GetWorld())
    {
      if (const UNetDriver* NetworkDriver = World->GetNetDriver())
      {
        if (UTutorialRepGraph* RepGraph = NetworkDriver->GetReplicationDriver<UTutorialRepGraph>())
        {
          RepGraph->SetTeamForPlayerController(GetPlayerController(), Team);
        }
      }
    }
  }
```
The result of this work is that each player can only see others in the same team. Great for a first step but only half way to where we want to be!

![Replication Graph3](rep3.jpg)

## Cleaning Up

Before we move on there are also a few functions to implement so that we clean up all state properly if an actor or connection is destroyed. First up is RemoveClientConnection which is pretty self explanatory, we just need to remove the connection graph from the TeamConnectionListMap.

```cs

void UTutorialRepGraph::RemoveClientConnection(UNetConnection* NetConnection)
{
  int32 ConnectionId = 0;
  bool bFound = false;
  
  auto UpdateList = [&](TArray<TObjectPtr<UNetReplicationGraphConnection>>& List)
  {
    for (int32 idx = 0; idx < List.Num(); ++idx)
    {
      UTutorialConnectionManager* ConnectionManager = Cast<UTutorialConnectionManager>(Connections[idx]);
      repCheck(ConnectionManager);

      if (ConnectionManager->NetConnection == NetConnection)
      {
        ensure(!bFound);

        // Remove the connection from the team node if the team is valid
        if (ConnectionManager->Team != -1)
        {
          TeamConnectionListMap.RemoveConnectionFromTeam(ConnectionManager->Team, ConnectionManager);
        }

        // Also remove it from the input list
        List.RemoveAtSwap(idx, 1, false);
        bFound = true;
      }
      else
      {
        ConnectionManager->ConnectionOrderNum = ConnectionId++;
      }
    }
  };

  UpdateList(Connections);
  UpdateList(PendingConnections);
}
```

Then there is RouteRemoveNetworkActorToNodes which is pretty much the reverse of RouteAddNetworkActorToNodes and just calls NotifyRemoveNetworkActor on each node that the actor was added to.

```cs

void UTutorialRepGraph::RouteRemoveNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo)
{
  if (ActorInfo.Class->IsChildOf(AGameStateBase::StaticClass()) || ActorInfo.Class->IsChildOf(APlayerState::StaticClass()))
  {
    AlwaysRelevantNode->NotifyRemoveNetworkActor(ActorInfo);
  }
  else if (const UTutorialConnectionManager* ConnectionManager = GetTutorialConnectionManagerFromActor(ActorInfo.GetActor()))
  {
    if (ActorInfo.Actor->bOnlyRelevantToOwner)
    {
      ConnectionManager->AlwaysRelevantForConnectionNode->NotifyRemoveNetworkActor(ActorInfo);
    }
    else
    {
      ConnectionManager->TeamConnectionNode->NotifyRemoveNetworkActor(ActorInfo);
    }
  }
  else if (ActorInfo.Actor->GetNetOwner())
  {
    PendingConnectionActors.Remove(ActorInfo.GetActor());
  }
}
```

Finally we can use ResetGameWorldState to clean up the graph when seamless travelling. This is only needed for seamless travel as non-seamless connections will be destroyed and recreated when travelling and should correctly clean up via RemoveClientConnection and RouteRemoveNetworkActorToNodes.


```cs

void UTutorialRepGraph::ResetGameWorldState()
{
  Super::ResetGameWorldState();

  PendingConnectionActors.Reset();
  PendingTeamRequests.Reset();
  
  auto EmptyConnectionNode = [](TArray<TObjectPtr<UNetReplicationGraphConnection>>& GraphConnections)
  {
    for (UNetReplicationGraphConnection* GraphConnection : GraphConnections)
    {
      if (const UTutorialConnectionManager* TutorialConnectionManager = Cast<UTutorialConnectionManager>(GraphConnection))
      {
        // Clear out all always relevant actors
        // Seamless travel means that the team connections will still be relevant due to the controllers not being destroyed
        TutorialConnectionManager->AlwaysRelevantForConnectionNode->NotifyResetAllNetworkActors();
      }
    }
  };

  EmptyConnectionNode(PendingConnections);
  EmptyConnectionNode(Connections);
}
```

## Extending The Replication Graph To Use Visibility