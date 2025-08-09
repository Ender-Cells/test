using System;
using System.Collections.Generic;
using FishNet.Broadcast;
using FishNet.Component.Observing;
using FishNet.Documenting;
using FishNet.Managing;
using FishNet.Managing.Logging;
using FishNet.Managing.Server;
using FishNet.Managing.Timing;
using FishNet.Managing.Transporting;
using FishNet.Object;
using FishNet.Serializing;
using FishNet.Transporting;
using GameKit.Dependencies.Utilities;
using UnityEngine;
using UnityEngine.SceneManagement;

namespace FishNet.Connection
{
	// Token: 0x0200020A RID: 522
	public class NetworkConnection : IResettable, IEquatable<NetworkConnection>
	{
		// Token: 0x06001109 RID: 4361 RVA: 0x0003C1A8 File Offset: 0x0003A3A8
		private void InitializeBuffer()
		{
			for (byte b = 0; b < 2; b += 1)
			{
				int lowestMTU = this.NetworkManager.TransportManager.GetLowestMTU(b);
				this._toClientBundles.Add(new PacketBundle(this.NetworkManager, lowestMTU, 0, DataOrderType.Default));
			}
		}

		// Token: 0x0600110A RID: 4362 RVA: 0x0003C1ED File Offset: 0x0003A3ED
		public void Broadcast<T>(T message, bool requireAuthenticated = true, Channel channel = Channel.Reliable) where T : struct, IBroadcast
		{
			if (!this.IsActive)
			{
				this.NetworkManager.LogError("Connection is not valid, cannot send broadcast.");
				return;
			}
			this.NetworkManager.ServerManager.Broadcast<T>(this, message, requireAuthenticated, channel);
		}

		// Token: 0x0600110B RID: 4363 RVA: 0x0003C21C File Offset: 0x0003A41C
		internal void SendToClient(byte channel, ArraySegment<byte> segment, bool forceNewBuffer = false, DataOrderType orderType = DataOrderType.Default)
		{
			if (this.Disconnecting)
			{
				return;
			}
			if (!this.IsActive)
			{
				this.NetworkManager.LogWarning(string.Format("Data cannot be sent to connection {0} because it is not active.", this.ClientId));
				return;
			}
			if ((int)channel >= this._toClientBundles.Count)
			{
				channel = 0;
			}
			this._toClientBundles[(int)channel].Write(segment, forceNewBuffer, orderType);
			this.ServerDirty();
		}

		// Token: 0x0600110C RID: 4364 RVA: 0x0003C287 File Offset: 0x0003A487
		internal bool GetPacketBundle(int channel, out PacketBundle packetBundle)
		{
			return PacketBundle.GetPacketBundle(channel, this._toClientBundles, out packetBundle);
		}

		// Token: 0x0600110D RID: 4365 RVA: 0x0003C296 File Offset: 0x0003A496
		private void ServerDirty()
		{
			bool serverDirtied = this._serverDirtied;
			this._serverDirtied = true;
			if (!serverDirtied)
			{
				this.NetworkManager.TransportManager.ServerDirty(this);
			}
		}

		// Token: 0x0600110E RID: 4366 RVA: 0x0003C2B8 File Offset: 0x0003A4B8
		internal void ResetServerDirty()
		{
			this._serverDirtied = false;
		}

		// Token: 0x17000195 RID: 405
		// (get) Token: 0x0600110F RID: 4367 RVA: 0x0003C2C1 File Offset: 0x0003A4C1
		// (set) Token: 0x06001110 RID: 4368 RVA: 0x0003C2C9 File Offset: 0x0003A4C9
		internal uint DisconnectingTick { get; private set; }

		// Token: 0x14000060 RID: 96
		// (add) Token: 0x06001111 RID: 4369 RVA: 0x0003C2D4 File Offset: 0x0003A4D4
		// (remove) Token: 0x06001112 RID: 4370 RVA: 0x0003C30C File Offset: 0x0003A50C
		public event Action<NetworkConnection, bool> OnLoadedStartScenes;

		// Token: 0x14000061 RID: 97
		// (add) Token: 0x06001113 RID: 4371 RVA: 0x0003C344 File Offset: 0x0003A544
		// (remove) Token: 0x06001114 RID: 4372 RVA: 0x0003C37C File Offset: 0x0003A57C
		public event Action<NetworkObject> OnObjectAdded;

		// Token: 0x14000062 RID: 98
		// (add) Token: 0x06001115 RID: 4373 RVA: 0x0003C3B4 File Offset: 0x0003A5B4
		// (remove) Token: 0x06001116 RID: 4374 RVA: 0x0003C3EC File Offset: 0x0003A5EC
		public event Action<NetworkObject> OnObjectRemoved;

		// Token: 0x17000196 RID: 406
		// (get) Token: 0x06001117 RID: 4375 RVA: 0x0003C421 File Offset: 0x0003A621
		// (set) Token: 0x06001118 RID: 4376 RVA: 0x0003C429 File Offset: 0x0003A629
		public NetworkManager NetworkManager { get; private set; }

		// Token: 0x06001119 RID: 4377 RVA: 0x0003C432 File Offset: 0x0003A632
		public bool LoadedStartScenes()
		{
			return this._loadedStartScenesAsServer || this._loadedStartScenesAsClient;
		}

		// Token: 0x0600111A RID: 4378 RVA: 0x0003C444 File Offset: 0x0003A644
		public bool LoadedStartScenes(bool asServer)
		{
			if (asServer)
			{
				return this._loadedStartScenesAsServer;
			}
			return this._loadedStartScenesAsClient;
		}

		// Token: 0x17000197 RID: 407
		// (get) Token: 0x0600111B RID: 4379 RVA: 0x0003C456 File Offset: 0x0003A656
		// (set) Token: 0x0600111C RID: 4380 RVA: 0x0003C45E File Offset: 0x0003A65E
		public int TransportIndex { get; internal set; } = -1;

		// Token: 0x17000198 RID: 408
		// (get) Token: 0x0600111D RID: 4381 RVA: 0x0003C467 File Offset: 0x0003A667
		// (set) Token: 0x0600111E RID: 4382 RVA: 0x0003C46F File Offset: 0x0003A66F
		public bool IsAuthenticated { get; private set; }

		// Token: 0x17000199 RID: 409
		// (get) Token: 0x0600111F RID: 4383 RVA: 0x0003C478 File Offset: 0x0003A678
		// (set) Token: 0x06001120 RID: 4384 RVA: 0x0003C480 File Offset: 0x0003A680
		[Obsolete("Use IsAuthenticated.")]
		public bool Authenticated
		{
			get
			{
				return this.IsAuthenticated;
			}
			set
			{
				this.IsAuthenticated = value;
			}
		}

		// Token: 0x1700019A RID: 410
		// (get) Token: 0x06001121 RID: 4385 RVA: 0x0003C489 File Offset: 0x0003A689
		public bool IsActive
		{
			get
			{
				return this.ClientId >= 0 && !this.Disconnecting;
			}
		}

		// Token: 0x1700019B RID: 411
		// (get) Token: 0x06001122 RID: 4386 RVA: 0x0003C49F File Offset: 0x0003A69F
		public bool IsValid
		{
			get
			{
				return this.ClientId >= 0;
			}
		}

		// Token: 0x1700019C RID: 412
		// (get) Token: 0x06001123 RID: 4387 RVA: 0x0003C4AD File Offset: 0x0003A6AD
		// (set) Token: 0x06001124 RID: 4388 RVA: 0x0003C4B5 File Offset: 0x0003A6B5
		public NetworkObject FirstObject { get; private set; }

		// Token: 0x06001125 RID: 4389 RVA: 0x0003C4C0 File Offset: 0x0003A6C0
		public void SetFirstObject(NetworkObject nob)
		{
			if (!this.Objects.Contains(nob))
			{
				string message = string.Format("FirstObject for {0} cannot be set to {1} as it's not within Objects for this connection.", this.ClientId, nob.name);
				this.NetworkManager.LogError(message);
				return;
			}
			this.FirstObject = nob;
		}

		// Token: 0x1700019D RID: 413
		// (get) Token: 0x06001126 RID: 4390 RVA: 0x0003C50B File Offset: 0x0003A70B
		// (set) Token: 0x06001127 RID: 4391 RVA: 0x0003C513 File Offset: 0x0003A713
		public HashSet<Scene> Scenes { get; private set; } = new HashSet<Scene>();

		// Token: 0x1700019E RID: 414
		// (get) Token: 0x06001128 RID: 4392 RVA: 0x0003C51C File Offset: 0x0003A71C
		// (set) Token: 0x06001129 RID: 4393 RVA: 0x0003C524 File Offset: 0x0003A724
		public bool Disconnecting { get; private set; }

		// Token: 0x1700019F RID: 415
		// (get) Token: 0x0600112A RID: 4394 RVA: 0x0003C52D File Offset: 0x0003A72D
		// (set) Token: 0x0600112B RID: 4395 RVA: 0x0003C535 File Offset: 0x0003A735
		public EstimatedTick PacketTick { get; private set; } = new EstimatedTick();

		// Token: 0x170001A0 RID: 416
		// (get) Token: 0x0600112C RID: 4396 RVA: 0x0003C53E File Offset: 0x0003A73E
		// (set) Token: 0x0600112D RID: 4397 RVA: 0x0003C546 File Offset: 0x0003A746
		public EstimatedTick LocalTick { get; private set; } = new EstimatedTick();

		// Token: 0x0600112E RID: 4398 RVA: 0x0003C550 File Offset: 0x0003A750
		public override bool Equals(object obj)
		{
			NetworkConnection networkConnection = obj as NetworkConnection;
			return networkConnection != null && networkConnection.ClientId == this.ClientId;
		}

		// Token: 0x0600112F RID: 4399 RVA: 0x0003C577 File Offset: 0x0003A777
		public bool Equals(NetworkConnection nc)
		{
			return nc != null && this.ClientId != -1 && nc.ClientId != -1 && (this == nc || this.ClientId == nc.ClientId);
		}

		// Token: 0x06001130 RID: 4400 RVA: 0x0003C5A6 File Offset: 0x0003A7A6
		public override int GetHashCode()
		{
			return this.ClientId;
		}

		// Token: 0x06001131 RID: 4401 RVA: 0x0003C5AE File Offset: 0x0003A7AE
		public static bool operator ==(NetworkConnection a, NetworkConnection b)
		{
			if (a == null && b == null)
			{
				return true;
			}
			if (a == null && b != null)
			{
				return false;
			}
			if (!(b == null))
			{
				return b.Equals(a);
			}
			return a.Equals(b);
		}

		// Token: 0x06001132 RID: 4402 RVA: 0x0003C5D8 File Offset: 0x0003A7D8
		public static bool operator !=(NetworkConnection a, NetworkConnection b)
		{
			return !(a == b);
		}

		// Token: 0x06001133 RID: 4403 RVA: 0x0003C5E4 File Offset: 0x0003A7E4
		[APIExclude]
		public NetworkConnection()
		{
		}

		// Token: 0x06001134 RID: 4404 RVA: 0x0003C674 File Offset: 0x0003A874
		[APIExclude]
		public NetworkConnection(NetworkManager manager, int clientId, int transportIndex, bool asServer)
		{
			this.Initialize(manager, clientId, transportIndex, asServer);
		}

		// Token: 0x06001135 RID: 4405 RVA: 0x0003C710 File Offset: 0x0003A910
		public override string ToString()
		{
			int clientId = this.ClientId;
			string arg = (this.NetworkManager != null) ? this.NetworkManager.TransportManager.Transport.GetConnectionAddress(clientId) : "Unset";
			return string.Format("Id [{0}] Address [{1}]", this.ClientId, arg);
		}

		// Token: 0x06001136 RID: 4406 RVA: 0x0003C768 File Offset: 0x0003A968
		private void Initialize(NetworkManager nm, int clientId, int transportIndex, bool asServer)
		{
			this.NetworkManager = nm;
			this.LocalTick.Initialize(nm.TimeManager, 0U, 0U, 0U);
			this.PacketTick.Initialize(nm.TimeManager, 0U, 0U, 0U);
			if (asServer)
			{
				this.ServerConnectionTick = nm.TimeManager.LocalTick;
			}
			this.TransportIndex = transportIndex;
			this.ClientId = clientId;
			this.PacketTick.Update(nm.TimeManager, 0U, EstimatedTick.OldTickOption.SetLastRemoteTick, true);
			this.Observers_Initialize(nm);
			this.Prediction_Initialize(nm, asServer);
			if (asServer)
			{
				this.InitializeBuffer();
				this.InitializePing();
			}
		}

		// Token: 0x06001137 RID: 4407 RVA: 0x0003C7FC File Offset: 0x0003A9FC
		internal void SetDisconnecting(bool value)
		{
			this.Disconnecting = value;
			if (this.Disconnecting)
			{
				this.DisconnectingTick = this.NetworkManager.TimeManager.LocalTick;
			}
		}

		// Token: 0x06001138 RID: 4408 RVA: 0x0003C824 File Offset: 0x0003AA24
		public void Disconnect(bool immediately)
		{
			if (!this.IsValid)
			{
				this.NetworkManager.LogWarning("Disconnect called on an invalid connection.");
				return;
			}
			if (this.Disconnecting)
			{
				this.NetworkManager.LogWarning(string.Format("ClientId {0} is already disconnecting.", this.ClientId));
				return;
			}
			this.SetDisconnecting(true);
			if (immediately)
			{
				this.NetworkManager.TransportManager.Transport.StopConnection(this.ClientId, true);
				return;
			}
			this.ServerDirty();
		}

		// Token: 0x06001139 RID: 4409 RVA: 0x0003C8A1 File Offset: 0x0003AAA1
		internal bool SetLoadedStartScenes(bool asServer)
		{
			bool result = !(asServer ? this._loadedStartScenesAsServer : this._loadedStartScenesAsClient);
			if (asServer)
			{
				this._loadedStartScenesAsServer = true;
			}
			else
			{
				this._loadedStartScenesAsClient = true;
			}
			Action<NetworkConnection, bool> onLoadedStartScenes = this.OnLoadedStartScenes;
			if (onLoadedStartScenes == null)
			{
				return result;
			}
			onLoadedStartScenes(this, asServer);
			return result;
		}

		// Token: 0x0600113A RID: 4410 RVA: 0x0003C8DC File Offset: 0x0003AADC
		internal void ConnectionAuthenticated()
		{
			this.IsAuthenticated = true;
		}

		// Token: 0x0600113B RID: 4411 RVA: 0x0003C8E5 File Offset: 0x0003AAE5
		internal void AddObject(NetworkObject nob)
		{
			if (!this.IsValid)
			{
				return;
			}
			this.Objects.Add(nob);
			if (this.Objects.Count == 1)
			{
				this.SetFirstObject();
			}
			Action<NetworkObject> onObjectAdded = this.OnObjectAdded;
			if (onObjectAdded == null)
			{
				return;
			}
			onObjectAdded(nob);
		}

		// Token: 0x0600113C RID: 4412 RVA: 0x0003C924 File Offset: 0x0003AB24
		internal void RemoveObject(NetworkObject nob)
		{
			if (!this.IsValid)
			{
				this.ClearObjects();
				return;
			}
			this.Objects.Remove(nob);
			if (nob == this.FirstObject)
			{
				this.SetFirstObject();
			}
			Action<NetworkObject> onObjectRemoved = this.OnObjectRemoved;
			if (onObjectRemoved == null)
			{
				return;
			}
			onObjectRemoved(nob);
		}

		// Token: 0x0600113D RID: 4413 RVA: 0x0003C972 File Offset: 0x0003AB72
		private void ClearObjects()
		{
			this.Objects.Clear();
			this.FirstObject = null;
		}

		// Token: 0x0600113E RID: 4414 RVA: 0x0003C988 File Offset: 0x0003AB88
		private void SetFirstObject()
		{
			if (this.Objects.Count == 0)
			{
				this.FirstObject = null;
				return;
			}
			using (HashSet<NetworkObject>.Enumerator enumerator = this.Objects.GetEnumerator())
			{
				if (enumerator.MoveNext())
				{
					NetworkObject firstObject = enumerator.Current;
					this.FirstObject = firstObject;
					this.Observers_FirstObjectChanged();
				}
			}
		}

		// Token: 0x0600113F RID: 4415 RVA: 0x0003C9F8 File Offset: 0x0003ABF8
		internal bool AddToScene(Scene scene)
		{
			return this.Scenes.Add(scene);
		}

		// Token: 0x06001140 RID: 4416 RVA: 0x0003CA06 File Offset: 0x0003AC06
		internal bool RemoveFromScene(Scene scene)
		{
			return this.Scenes.Remove(scene);
		}

		// Token: 0x06001141 RID: 4417 RVA: 0x0003CA14 File Offset: 0x0003AC14
		public void ResetState()
		{
			MatchCondition.RemoveFromMatchesWithoutRebuild(this, this.NetworkManager);
			foreach (PacketBundle packetBundle in this._toClientBundles)
			{
				packetBundle.Dispose();
			}
			this._toClientBundles.Clear();
			this.ServerConnectionTick = 0U;
			this.PacketTick.Reset();
			this.LocalTick.Reset();
			this.TransportIndex = -1;
			this.ClientId = -1;
			this.ClearObjects();
			this.IsAuthenticated = false;
			this.HasSentVersion = false;
			this.NetworkManager = null;
			this._loadedStartScenesAsClient = false;
			this._loadedStartScenesAsServer = false;
			this.SetDisconnecting(false);
			this.Scenes.Clear();
			this.PredictedObjectIds.Clear();
			this.ResetPingPong();
			this.Observers_Reset();
			this.Prediction_Reset();
		}

		// Token: 0x06001142 RID: 4418 RVA: 0x00003E84 File Offset: 0x00002084
		public void InitializeState()
		{
		}

		// Token: 0x06001143 RID: 4419 RVA: 0x0003CB00 File Offset: 0x0003AD00
		private void Observers_FirstObjectChanged()
		{
			this.UpdateHashGridPositions(true);
		}

		// Token: 0x06001144 RID: 4420 RVA: 0x0003CB09 File Offset: 0x0003AD09
		private void Observers_Initialize(NetworkManager nm)
		{
			nm.TryGetInstance<HashGrid>(out this._hashGrid);
		}

		// Token: 0x06001145 RID: 4421 RVA: 0x0003CB18 File Offset: 0x0003AD18
		internal void UpdateHashGridPositions(bool force)
		{
			if (this._hashGrid == null)
			{
				return;
			}
			float unscaledTime = Time.unscaledTime;
			if (!force && unscaledTime < this._nextHashGridUpdateTime)
			{
				return;
			}
			this._nextHashGridUpdateTime = unscaledTime + 1f;
			if (this.FirstObject == null)
			{
				this.HashGridEntry = HashGrid.EmptyGridEntry;
				this._hashGridPosition = HashGrid.UnsetGridPosition;
				return;
			}
			Vector2Int hashGridPosition = this._hashGrid.GetHashGridPosition(this.FirstObject);
			if (hashGridPosition != this._hashGridPosition)
			{
				this._hashGridPosition = hashGridPosition;
				this.HashGridEntry = this._hashGrid.GetGridEntry(hashGridPosition);
			}
		}

		// Token: 0x06001146 RID: 4422 RVA: 0x0003CBB2 File Offset: 0x0003ADB2
		private void Observers_Reset()
		{
			this._hashGrid = null;
			this._hashGridPosition = HashGrid.UnsetGridPosition;
			this._nextHashGridUpdateTime = 0f;
		}

		// Token: 0x06001147 RID: 4423 RVA: 0x0003CBD4 File Offset: 0x0003ADD4
		private void InitializePing()
		{
			float num = (float)this.NetworkManager.TimeManager.PingInterval * 0.85f;
			this._requiredPingTicks = this.NetworkManager.TimeManager.TimeToTicks((double)num, TickRounding.RoundDown);
		}

		// Token: 0x06001148 RID: 4424 RVA: 0x0003CC12 File Offset: 0x0003AE12
		private void ResetPingPong()
		{
			this._lastPingTick = 0U;
		}

		// Token: 0x06001149 RID: 4425 RVA: 0x0003CC1C File Offset: 0x0003AE1C
		internal bool CanPingPong()
		{
			TimeManager timeManager = (this.NetworkManager == null) ? InstanceFinder.TimeManager : this.NetworkManager.TimeManager;
			if (timeManager.LowFrameRate)
			{
				return false;
			}
			uint tick = timeManager.Tick;
			uint num = tick - this._lastPingTick;
			this._lastPingTick = tick;
			return num >= this._requiredPingTicks;
		}

		// Token: 0x170001A1 RID: 417
		// (get) Token: 0x0600114A RID: 4426 RVA: 0x0003CC75 File Offset: 0x0003AE75
		// (set) Token: 0x0600114B RID: 4427 RVA: 0x0003CC7D File Offset: 0x0003AE7D
		public EstimatedTick ReplicateTick { get; private set; } = new EstimatedTick();

		// Token: 0x0600114C RID: 4428 RVA: 0x00003E84 File Offset: 0x00002084
		internal void Prediction_Initialize(NetworkManager manager, bool asServer)
		{
		}

		// Token: 0x0600114D RID: 4429 RVA: 0x0003CC88 File Offset: 0x0003AE88
		internal void WriteState(PooledWriter data)
		{
			if (this.IsLocalClient)
			{
				return;
			}
			TimeManager timeManager = this.NetworkManager.TimeManager;
			TransportManager transportManager = this.NetworkManager.TransportManager;
			if ((ulong)(this.IsLocalClient ? 0U : this.PacketTick.LocalTickDifference(timeManager)) > (ulong)((long)(timeManager.TickRate * 5)))
			{
				return;
			}
			int lowestMTU = transportManager.GetLowestMTU(1);
			int count = this.PredictionStateWriters.Count;
			Channel channel = Channel.Unreliable;
			if (count > 0)
			{
				transportManager.CheckSetReliableChannel(data.Length + this.PredictionStateWriters[count - 1].Length, ref channel);
			}
			PooledWriter pooledWriter;
			if (count == 0 || channel == Channel.Reliable)
			{
				pooledWriter = WriterPool.Retrieve(lowestMTU);
				this.PredictionStateWriters.Add(pooledWriter);
				pooledWriter.Skip(10);
			}
			else
			{
				pooledWriter = this.PredictionStateWriters[count - 1];
			}
			pooledWriter.WriteArraySegment(data.GetArraySegment());
		}

		// Token: 0x0600114E RID: 4430 RVA: 0x0003CD5C File Offset: 0x0003AF5C
		internal void StorePredictionStateWriters()
		{
			for (int i = 0; i < this.PredictionStateWriters.Count; i++)
			{
				WriterPool.Store(this.PredictionStateWriters[i]);
			}
			this.PredictionStateWriters.Clear();
		}

		// Token: 0x0600114F RID: 4431 RVA: 0x0003CD9B File Offset: 0x0003AF9B
		internal void SetReplicateTick(uint value, EstimatedTick.OldTickOption oldTickOption = EstimatedTick.OldTickOption.Discard)
		{
			this.ReplicateTick.Update(value, oldTickOption, true);
		}

		// Token: 0x06001150 RID: 4432 RVA: 0x0003CDAC File Offset: 0x0003AFAC
		private void Prediction_Reset()
		{
			this.StorePredictionStateWriters();
			this.ReplicateTick.Reset();
		}

		// Token: 0x170001A2 RID: 418
		// (get) Token: 0x06001151 RID: 4433 RVA: 0x0003CDBF File Offset: 0x0003AFBF
		public bool IsHost
		{
			get
			{
				return !(this.NetworkManager == null) && this.NetworkManager.IsServerStarted && this == this.NetworkManager.ClientManager.Connection;
			}
		}

		// Token: 0x170001A3 RID: 419
		// (get) Token: 0x06001152 RID: 4434 RVA: 0x0003CDF6 File Offset: 0x0003AFF6
		public bool IsLocalClient
		{
			get
			{
				return !(this.NetworkManager == null) && this.NetworkManager.ClientManager.Connection == this;
			}
		}

		// Token: 0x06001153 RID: 4435 RVA: 0x0003CE1E File Offset: 0x0003B01E
		public string GetAddress()
		{
			if (!this.IsValid)
			{
				return string.Empty;
			}
			if (this.NetworkManager == null)
			{
				return string.Empty;
			}
			return this.NetworkManager.TransportManager.Transport.GetConnectionAddress(this.ClientId);
		}

		// Token: 0x06001154 RID: 4436 RVA: 0x0003CE5D File Offset: 0x0003B05D
		public void Kick(KickReason kickReason, LoggingType loggingType = LoggingType.Common, string log = "")
		{
			this.NetworkManager.ServerManager.Kick(this, kickReason, loggingType, log);
		}

		// Token: 0x06001155 RID: 4437 RVA: 0x0003CE73 File Offset: 0x0003B073
		public void Kick(Reader reader, KickReason kickReason, LoggingType loggingType = LoggingType.Common, string log = "")
		{
			this.NetworkManager.ServerManager.Kick(this, reader, kickReason, loggingType, log);
		}

		// Token: 0x040008A0 RID: 2208
		private List<PacketBundle> _toClientBundles = new List<PacketBundle>();

		// Token: 0x040008A1 RID: 2209
		private bool _serverDirtied;

		// Token: 0x040008A3 RID: 2211
		internal Queue<int> PredictedObjectIds = new Queue<int>();

		// Token: 0x040008A4 RID: 2212
		internal bool HasSentVersion;

		// Token: 0x040008A5 RID: 2213
		internal uint ServerConnectionTick;

		// Token: 0x040008AC RID: 2220
		public int ClientId = -1;

		// Token: 0x040008AD RID: 2221
		public HashSet<NetworkObject> Objects = new HashSet<NetworkObject>();

		// Token: 0x040008B1 RID: 2225
		public object CustomData;

		// Token: 0x040008B4 RID: 2228
		private bool _loadedStartScenesAsServer;

		// Token: 0x040008B5 RID: 2229
		private bool _loadedStartScenesAsClient;

		// Token: 0x040008B6 RID: 2230
		public const int UNSET_CLIENTID_VALUE = -1;

		// Token: 0x040008B7 RID: 2231
		public const int MAXIMUM_CLIENTID_VALUE = 2147483647;

		// Token: 0x040008B8 RID: 2232
		public const int MAXIMUM_CLIENTID_WITHOUT_SIMULATED_VALUE = 2147483646;

		// Token: 0x040008B9 RID: 2233
		public const int SIMULATED_CLIENTID_VALUE = 2147483647;

		// Token: 0x040008BA RID: 2234
		public const int CLIENTID_UNCOMPRESSED_RESERVE_LENGTH = 4;

		// Token: 0x040008BB RID: 2235
		internal GridEntry HashGridEntry = HashGrid.EmptyGridEntry;

		// Token: 0x040008BC RID: 2236
		private HashGrid _hashGrid;

		// Token: 0x040008BD RID: 2237
		private float _nextHashGridUpdateTime;

		// Token: 0x040008BE RID: 2238
		private Vector2Int _hashGridPosition = HashGrid.UnsetGridPosition;

		// Token: 0x040008BF RID: 2239
		private uint _lastPingTick;

		// Token: 0x040008C0 RID: 2240
		private uint _requiredPingTicks;

		// Token: 0x040008C1 RID: 2241
		private const byte EXCESSIVE_PING_LIMIT = 10;

		// Token: 0x040008C3 RID: 2243
		internal List<PooledWriter> PredictionStateWriters = new List<PooledWriter>();
	}
}
