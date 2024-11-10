# Bed Desync Exploit in Minecraft 1.9.4

The **Bed Desync Exploit** in Minecraft 1.9.4 allows players to bypass server-client synchronization when interacting with beds. In certain circumstances, this desynchronization causes **ghost blocks** to appear, enabling players to break **claimed faction blocks** locally. This exploit takes advantage of the game's failure to update the world state properly, allowing players to manipulate block interactions in ways that would normally be restricted.

### Credits

- **Adwayz** and **kailleblo78** for discovering the exploit.

### Video Demonstration

For a visual demonstration of the exploit in action, check out the video below:

[**Bed Desync Exploit Demonstration**](https://www.youtube.com/watch?v=PB-J5z7Db88)

---

## Why does this work?

The **Bed Desync Exploit** relies on a series of **client-server desynchronization** packets during the sleeping interaction with a bed. Here's how the exploit can be reproduced:

### Client to Server:

1. **Use Entity (0x0A)**: The client sends a packet to interact with the bed, initiating the sleeping action.

### Server to Client:

1. **Entity Metadata (0x44)**: The server updates the player’s pose to "sleeping."
2. **Block Entity Data (0x0A)**: The server updates the bed's state to "occupied."
3. **Player Position and Look (0x36)**: The server adjusts the player’s position and angle to fit the sleeping pose.

### Exploit Conditions:

When **network latency or a netlimiter is used**, the client never receives the final packet confirming that the bed interaction is complete, and the bed is marked as "occupied." As a result:
- **The bed block is broken locally** by the client before the final packet arrives.
- When the packet finally arrives, it tries to update a block that no longer exists (since it's now air), causing the **ghost block** behavior.

### Error Breakdown

Due to the desynchronization, an error occurs in the client:

```plaintext
[Client Thread/FATAL]: Error executing task
java.lang.IllegalArgumentException: Cannot get property WZi{valueClass=class x.wTc, name='facing', allowedValues=[north, south, west, east]} as it does not exist in tzL{block=minecraft:air, properties=[]}
        at x.izN.W(S:61) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.Cni.v(S:355) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.aCs.I(S:732) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.iRP.I(S:10) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.iRP.I(S:4) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.aCs.I(S:689) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.Cxi.I(S:1) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.inr.I(S:8) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.xMa.aG(S:1974) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.xMa.bp(S:1629) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.xMa.ak(S:1368) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at x.RmP.W(S:28) ~[4ac66e54397448c781b353f5ec81e125:1.0-SNAPSHOT-f4ce770-2152]
        at java.lang.Thread.run(Thread.java:748) [?:1.8.0_251]
```


The error occurs because the server is attempting to set properties (such as the `facing` direction) on a block that no longer exists locally (now represented as `minecraft:air`). This results in the **IllegalArgumentException** because the air block does not have any properties like `facing`.

### Code Explanation

The underlying issue can be traced in the **`EntityPlayer.java`** file. In the method `trySleep`, the server attempts to process the bed interaction and set the player's position. If the block at the bed's location is unloaded or otherwise unavailable, the server tries to interact with the block properties, but the client has already removed it locally.

Specifically, this line of code fails due to the block being replaced by air:
```java
EnumFacing enumfacing = (EnumFacing) this.worldObj.getBlockState(bedLocation).getValue(BlockHorizontal.FACING);
```



Here's the relevant code:

```java
public EntityPlayer.SleepResult trySleep(BlockPos bedLocation)
    {
        if (!this.worldObj.isRemote)
        {
            if (this.isPlayerSleeping() || !this.isEntityAlive())
            {
                return EntityPlayer.SleepResult.OTHER_PROBLEM;
            }

            if (!this.worldObj.provider.isSurfaceWorld())
            {
                return EntityPlayer.SleepResult.NOT_POSSIBLE_HERE;
            }

            if (this.worldObj.isDaytime())
            {
                return EntityPlayer.SleepResult.NOT_POSSIBLE_NOW;
            }

            if (Math.abs(this.posX - (double)bedLocation.getX()) > 3.0D || Math.abs(this.posY - (double)bedLocation.getY()) > 2.0D || Math.abs(this.posZ - (double)bedLocation.getZ()) > 3.0D)
            {
                return EntityPlayer.SleepResult.TOO_FAR_AWAY;
            }

            double d0 = 8.0D;
            double d1 = 5.0D;
            List<EntityMob> list = this.worldObj.<EntityMob>getEntitiesWithinAABB(EntityMob.class, new AxisAlignedBB((double)bedLocation.getX() - d0, (double)bedLocation.getY() - d1, (double)bedLocation.getZ() - d0, (double)bedLocation.getX() + d0, (double)bedLocation.getY() + d1, (double)bedLocation.getZ() + d0));

            if (!list.isEmpty())
            {
                return EntityPlayer.SleepResult.NOT_SAFE;
            }
        }

        if (this.isRiding())
        {
            this.dismountRidingEntity();
        }

        this.setSize(0.2F, 0.2F);

        if (this.worldObj.isBlockLoaded(bedLocation))
        {
            EnumFacing enumfacing = (EnumFacing)this.worldObj.getBlockState(bedLocation).getValue(BlockHorizontal.FACING);
            float f = 0.5F;
            float f1 = 0.5F;

            switch (enumfacing)
            {
                case SOUTH:
                    f1 = 0.9F;
                    break;

                case NORTH:
                    f1 = 0.1F;
                    break;

                case WEST:
                    f = 0.1F;
                    break;

                case EAST:
                    f = 0.9F;
            }

            this.setRenderOffsetForSleep(enumfacing);
            this.setPosition((double)((float)bedLocation.getX() + f), (double)((float)bedLocation.getY() + 0.6875F), (double)((float)bedLocation.getZ() + f1));
        }
        else
        {
            this.setPosition((double)((float)bedLocation.getX() + 0.5F), (double)((float)bedLocation.getY() + 0.6875F), (double)((float)bedLocation.getZ() + 0.5F));
        }

        this.sleeping = true;
        this.sleepTimer = 0;
        this.playerLocation = bedLocation;
        this.motionX = this.motionZ = this.motionY = 0.0D;

        if (!this.worldObj.isRemote)
        {
            this.worldObj.updateAllPlayersSleepingFlag();
        }

        return EntityPlayer.SleepResult.OK;
    }
```
