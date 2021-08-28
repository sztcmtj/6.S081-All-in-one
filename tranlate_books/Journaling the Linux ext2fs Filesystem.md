# Journaling the Linux ext2fs Filesystem

# 摘要

本文描述了为Linux ext2fs文件系统设计和实现事务元数据日志的工作进展。我们回顾了崩溃后恢复文件系统的问题，并描述了一种旨在通过向文件系统添加事务日志来提高ext2fs崩溃恢复速度和可靠性的设计。

# 介绍

文件系统是任何现代操作系统的核心部分，人们期望它既快速又非常可靠。但是，由于硬件、软件或电源故障，问题仍然存在，机器可能会意外停机。

在一次意料之外的重启后，系统可能需要一些时间才能恢复文件系统的一致性状态。随着磁盘大小的增长，这一时间可能会成为一个严重的问题，在扫描、检查和修复磁盘时，系统会离线一小时或更长时间。尽管磁盘驱动器的速度每年都在加快，但与容量的巨大增长相比，这一速度的增长并不明显。不幸的是，在使用传统的文件系统检查技术时，磁盘容量每增加一倍，恢复时间就会增加一倍。

在系统可用性很重要的情况下，这可能是无法节省的时间，因此需要一种机制，以避免每次机器重新启动时都需要昂贵的恢复阶段。

# 什么是文件系统

对于任何文件系统，我们都需要什么功能？文件系统所服务的操作系统有明确的要求。文件系统对应用程序的表现方式是：一个操作系统通常需要遵守某些约定的文件名，并且文件具有某些以特定方式解释的属性。

然而，文件系统的许多内部方面没有那么受约束，文件系统实现者可以在一定程度上自由地设计这些方面。磁盘上数据的布局（或者，如果文件系统不是本地的，它的网络协议）、内部缓存的细节以及用于调度磁盘IO的算法——在不违反文件系统应用程序接口规范的前提下，这些都是可以改变的。

我们可能选择一种而不是另一种设计的原因有很多。与旧文件系统的兼容性可能是一个问题：例如，Linux提供了一个UMSDOS文件系统，它在标准MSDOS磁盘文件结构的基础上实现了POSIX文件系统的语义学。

当试图解决Linux上文件系统恢复时间过长的问题时，我们牢记许多目标：

- 使用新文件系统不会严重影响性能；
- 不得破坏与现有应用程序的兼容性
- 文件系统的可靠性不得以任何方式受到损害。

# 文件系统可靠性

当我们谈论文件系统的可靠性时，有许多问题利害攸关。就本特定项目而言，我们主要关心的是恢复崩溃文件系统内容的可靠性，我们可以确定其中的几个方面：

**保持（Preservation）**：崩溃前磁盘上稳定的数据永远不会被损坏。显然，崩溃时正在写入的文件不能保证完全完好无损，但是恢复系统不能碰磁盘上已经安全的任何文件。

**可预测性（Predictability）**：我们必须恢复的故障模式应该是可预测的，以便我们可靠地恢复。

**原子性（Atomicity）**：许多文件系统操作需要大量独立的IO来完成。一个很好的例子是将文件从一个目录重命名到另一个目录。如果这样的文件系统操作在磁盘上完全完成，或者在恢复完成后完全撤销，恢复就是原子性的。（对于重命名的例子，恢复应该在崩溃后保留提交给磁盘的旧文件名或新文件名，但不能两者都保留。）

# 现有实现

Linux ext2fs文件系统提供了保留恢复（preserving recovery），但它是非原子的，不可预测。事实上，可预测性比乍一看要复杂得多。为了能够在崩溃后进行可预测的清理，恢复阶段必须能够确定文件系统在遇到表现为一次不完整操作的磁盘不一致性时试图做什么。通常，这要求在一次涉及磁盘上多个块更改的更新操作时，文件系统必须以可预测的顺序写入磁盘。

实现磁盘写入之间的这种排序有许多方法。最简单的方法是简单地等待第一次写入完成，然后再将下一次写入提交给设备驱动程序——“同步元数据更新（synchronous metadata update）”方法。这是BSD快速文件系统采取的方法，出现在4.2BSD中，它启发了随后的许多Unix文件系统，包括ext2fs。

然而，同步元数据更新的一大缺点是它的性能。如果文件系统操作要求我们等待磁盘IO完成，那么我们就不能将多个文件系统更新批处理成单个磁盘写入。例如，如果我们在磁盘上的同一个目录块中创建十几个目录项，那么同步更新需要我们将该块写回磁盘十几次。

有一些方法可以解决这个性能问题。一种保持磁盘写入顺序而不实际等待IO完成的方法是在内存中的磁盘缓冲区之间保持顺序，并确保当我们最终去写回数据时，在一个块的所有前置块都安全地写回磁盘前，我们永远都不会写该块，——“延迟有序写入”技术。

延迟有序写入的一个复杂性是很容易陷入缓存缓冲区之间存在循环依赖的情况。例如，如果我们试图在两个目录之间重命名一个文件，同时将第二个目录中的另一个文件重命名至第一个目录，那么我们最终会遇到两个目录块相互依赖的情况：两个目录块都在等待对方写回磁盘，最终都不能写入。

Ganger的“软更新”机制巧妙地避开了这个问题，当我们第一次尝试将缓冲区写入磁盘时，如果这些更新仍然有未完成的依赖关系，我们会有选择地回滚缓冲区中的特定更新。丢失的更新将在所有的依赖关系都得到满足后恢复。这使得我们可以在有循环依赖关系时以我们选择的任何顺序写入缓冲区。软更新机制已经被FreeBSD采用，并将作为他们下一个主要内核版本的一部分提供。

然而，所有这些方法都有一个共同的问题。尽管它们确保磁盘的状态在文件系统操作过程中一直处于可预测的状态，但恢复过程仍然必须扫描整个磁盘，以便找到和修复任何未完成的操作。恢复变得更加可靠，但不一定更快。

然而，在不牺牲可靠性和可预测性的情况下快速恢复文件系统是可能的。这通常由保证文件系统更新原子完成的文件系统来完成（在这样的系统中，单个文件系统更新通常被称为事务）。原子更新背后的基本原则是文件系统可以将整批新数据写入磁盘，但是这些更新在磁盘上进行最终提交更新之前不会生效。如果提交涉及到对磁盘的单个块的写入，那么崩溃只能导致两种情况：要么提交记录已经写入磁盘，在这种情况下，所有提交的文件系统操作都可以假设是完整的，并且在磁盘上是一致的；要么提交记录丢失，这种情况下，由于在崩溃时部分尚未提交的更新仍未完成，我们必须忽略任何其他写入操作。这自然需要文件系统更新，以将更新数据的新旧内容保存在磁盘上的某个地方，直到提交。

有许多方法可以实现这一点。在某些情况下，文件系统将更新数据的新副本保存在与旧副本不同的位置，并最终在更新提交到磁盘后重用旧空间。网络设备的WAFL文件系统是这样工作的，维护一个文件系统数据树，它可以通过将树节点复制到新的位置，然后更新树根部的单个磁盘块来进行原子更新。（注：在WAFL中，如果它也修改一个数据，他可能不管以前的数据的位置，直接把新数据与新校验写到新的位置，之后更改指针，告诉文件系统说，新的数据在这里，而不是原来那里了。）

日志结构化文件系统通过将所有文件系统数据——包括文件内容和元数据——以连续流（“日志”）写入磁盘来实现相同的目的。使用这种方案查找一段数据的位置可能比在传统文件系统中更复杂，但是日志有一个很大的优势，那就是在日志中放置标记相对容易，以指示直到某个点的所有数据都已提交并在磁盘上保持一致。写入这样的文件系统也特别快，因为日志的性质使得大多数写入发生在没有磁盘查找的连续流中。许多文件系统都是基于这种设计编写的，包括Sprite LFS和Berkeley LFS。Linux上也有一个原型LFS实现。

最后，还有一类原子更新的文件系统，它将新版本写入磁盘上的单独位置，并在更新提交前保留旧版本和更新不完整的新版本。提交后，文件系统可以自由地将更新磁盘块的新版本写回磁盘上的原始位置。

这是日志记录journaling（有时称为日志增强版log enhanced）文件系统的工作方式。当磁盘上的元数据被更新时，更新被记录在磁盘上用作日志保留的单独区域中。完成的文件系统事务将提交记录添加到日志中，只有在提交安全地存储在磁盘上后，文件系统才能将元数据写回其原始位置。事务是原子的，因为我们总是可以在崩溃后根据日志是否包含事务的提交记录撤销事务（丢弃日志中的新数据）或重做事务（将日志副本复制回原始副本）。许多现代文件系统采用了这种设计的变体。

# 为Linux设计一个新的文件系统

Linux新文件系统设计背后的主要动机是消除崩溃后大型文件系统恢复时间。出于这个原因，我们选择了文件系统日志计划作为这项工作的基础。日志实现了快速的文件系统恢复，因为我们知道在任何时候，磁盘上可能不一致的所有数据都必须记录在日志中。因此，可以通过扫描日志并将所有提交的数据复制回主文件系统区域来实现文件系统恢复。这很快，因为日志通常比完整的文件系统小得多。它只需要足够记录几秒钟的未提交更新的容量。

选择日志记录还有另一个重要优势。日志记录文件系统不同于传统文件系统，因为它将临时数据保存在一个新的位置，独立于磁盘上的永久数据和元数据。正因为如此，这样的文件系统并不要求永久数据必须以任何特定的方式存储。特别是，ext2fs文件系统的磁盘结构很有可能在新文件系统中使用，现有的ext2fs代码也很有可能用作日志记录版本的基础。

因此，我们不是在为Linux设计一个新的文件系统。相反，我们在现有的ext2fs中添加了一个新的特性——事务性文件系统日志记录

# 事务剖析

当考虑日志文件系统时，一个核心概念是事务，对应于文件系统的单个更新。应用程序发出的任何单个文件系统请求都会产生一个事务，并且包含该请求产生的所有更改的元数据。例如，对文件的写入将导致对文件在磁盘上的索引节点中的修改时间戳的更新，如果文件被写操作扩展，还可能更新长度信息和块映射信息。配额信息、空闲磁盘空间和记录使用块的位图都必须更新，如果给文件分配新的块，所有这些都必须记录在事务中。

在事务中还有一个我们必须注意的隐藏操作。事务还包括读取文件系统的现有内容，这在事务之间强加了顺序。修改磁盘上块的事务不能在读取新数据并根据读取的内容更新磁盘的事务之后提交。即使两个事务从来没有尝试写回相同的块，依赖性也是存在的——想象一个事务从目录中的一个块中删除文件名，另一个事务将相同的文件名插入到不同的块中。这两个操作在它们写入的块中可能不会重叠，但是第二个操作只有在第一个操作成功后才有效（违反这一操作将导致重复的目录条目）。

最后，除了元数据更新之间的排序之外，还有一个排序要求。在我们提交将新块分配给文件的事务之前，我们必须绝对确保事务创建的所有数据块实际上都已写入磁盘（我们称这些数据块为依赖数据*dependent data*）。忽略此要求实际上不会损害文件系统元数据的完整性，但它可能会导致新文件崩溃恢复后仍包含以前的文件内容，这是一个安全风险，也是一个一致性问题。

# 事务合并

日志文件系统中使用的许多术语和技术来自数据库世界，日志是确保复杂事务原子提交的标准机制。然而，传统数据库事务和文件系统之间有许多不同之处，其中一些允许我们大大简化事情。

两个最大的区别是文件系统没有事务中止，所有文件系统事务都相对短暂。而在数据库中，我们有时想中途中止事务，丢弃我们迄今为止所做的任何更改，在ext2fs中情况并非如此——当我们开始对文件系统进行任何更改时，我们已经检查了更改是否可以合法完成。在我们开始写入更改之前中止事务（例如，如果一个创建文件操作找到一个相同名称的现有文件，它可能会中止）不会带来任何问题，因为在这种情况下，我们可以简单地提交事务而不做任何更改，并实现相同的效果。

第二个区别——文件系统事务存在期很短——这很重要，因为这意味着我们可以极大地简化事务之间的依赖关系。如果我们必须满足一些非常长期的事务，那么我们需要允许事务以任何顺序独立提交，只要它们彼此不冲突，否则一个停滞不前的事务可能会拖累整个系统。然而，如果所有事务都足够快，那么我们可以要求事务以严格的顺序提交到磁盘，而不会明显损害性能。

通过这个观察，我们可以对事务模型进行简化，从而大大降低实现的复杂性，同时提高性能。与为每个文件系统更新创建单独的事务不同，我们只是经常创建一个新事务，并允许所有文件系统服务调用将它们的更新添加到单个系统范围的复合事务中。

这种机制有一个很大的优点。因为复合事务中的所有操作都将一起提交到日志中，所以我们不必为任何经常更新的元数据块编写单独的副本。特别是，这有助于创建新文件等操作，在这些操作中，对文件的每次写入都会导致文件被扩展，从而连续更新相同的配额、位图块和索引节点块。在复合事务的生命周期中，任何多次更新的块只需要提交到磁盘一次。

关于何时提交当前复合事务并启动新事务的决定是一个应该由用户控制的策略决定，因为它涉及到影响系统性能的权衡。提交等待的时间越长，可以在日志中合并的文件系统操作就越多，因此从长远来看需要的IO操作就越少。然而，更长的提交占用了大量的内存和磁盘空间，并在崩溃发生时留下了更大的更新丢失窗口。它们还可能导致磁盘活动的骤变，从而使文件系统响应时间难以预测。

# 磁盘表示

磁盘上记录的ext2fs文件系统的布局将与现有的ext2fs内核完全兼容。传统的UNIX文件系统通过将每个文件与磁盘上唯一编号的inode关联起来，将数据存储在磁盘上，而ext2fs设计已经包含了许多保留的inode编号。我们使用其中一个保留索引节点来存储文件系统日志，并且在所有其他方面，文件系统都将与现有的Linux内核兼容。现有的ext2fs设计包括一组兼容性位图，其中可以设置位来指示文件系统是否使用特定扩展。通过为日志扩展分配一个新的兼容性位，我们可以确保即使旧内核能够成功挂载一个新的、日志记录的ext2fs文件系统，它们也不会被允许以任何方式写入文件系统。

# 文件系统日志的格式

日志文件的工作很简单：它在我们提交事务的过程中记录文件系统元数据块的新内容。日志的唯一其他要求是我们必须能够原子地提交它包含的事务。

我们向日志写入三种不同类型的数据块：元数据块、描述符块和头块（metadata, descriptor and header blocks）。

日志元数据块包含由事务更新的单个文件系统元数据块的全部内容。这意味着，无论我们对文件系统元数据块做了多么小的更改，我们都必须写出整个日志块来记录更改。然而，由于两个原因，这一成本相对较低：

- 无论如何，日志写入非常快，因为对日志的大多数写入都是顺序的，我们可以很容易地将日志IO批处理成大型集群，磁盘控制器可以有效地处理这些集群；
- 通过将更改后的元数据缓冲区的全部内容从文件系统缓存写入日志，我们可以避免在日志代码中执行大量CPU工作。

Linux内核已经为我们提供了一种非常有效的机制，可以将buffer cache中现有块的内容写到磁盘上的不同位置。buffer cache中的每个缓冲区都由一个名为`buffer_head`的结构体描述，该结构体包括缓冲区的数据要写到哪个磁盘块的信息。如果我们想将整个缓冲区块在不干扰`buffer_head`的情况下写入新位置，我们可以简单地创建一个新的临时`buffer_head`，将旧的描述复制到其中，然后编辑临时`buffer_head`中的设备块编号字段，以指向日志文件中的块。然后，我们可以将临时`buffer_head`直接提交给设备IO系统，并在IO完成后丢弃它。

描述符块是描述其他日志元数据块的日志块，每当我们要将元数据块写出到日志时，我们需要记录下元数据通常安置在哪些磁盘块，这样恢复机制就可以将元数据复制回主文件系统中。在日志中的每一组元数据块之前都会写出一个描述符块，其中包含要写入的元数据块的数量加上它们的磁盘块号。

描述符块和元数据块都按顺序写入日志，每当我们运行超过末尾时，都会从日志的开头重新开始。在任何时候，我们都维护当前的日志头（最后写入的块的块号）和尾部（日志中尚未取消固定的最老的块，如下所述）。每当我们用完日志空间时——日志的头部已经循环回来并赶上了尾部——我们会停止新的日志写入，直到日志的尾部被清理干净，以释放更多的空间。

最后，日志文件包含一些位于固定位置的头块。这些头块记录了日志的当前头部和尾部，加上序列号。在恢复时，头块被扫描以找到序列号最高的块，当我们在恢复过程中扫描日志时，我们只是运行从尾部到头部的所有日志块，就像头块中记录的那样。

# 日志的提交和检查点

在某个时候，要么是因为上次提交后我们已经等了足够长的时间，要么是因为日志中的空间不足，我们希望将未完成的文件系统更新作为一个新的复合事务提交到日志中。

复合事务被完全提交后，我们仍然没有完成它。我们需要跟踪记录在事务中的元数据缓冲区，这样我们就可以注意到它们何时被写回磁盘上的主位置。

回想一下，当我们提交事务时，新更新的文件系统块位于日志中，但尚未同步回磁盘上的永久家块（家块就是写入操作对应的磁盘中文件系统对应的块，我们需要保持旧块的这种不同步，以防在提交日志之前崩溃）。一旦提交了日志，磁盘上的旧版本就不再重要，我们可以在闲暇时将缓冲区写回它们的主位置。但是，在同步完这些缓冲区之前，我们不能删除日志中数据的副本。

要完全提交并完成事务的检查点，我们将经历以下阶段：

1. 关闭事务。在此刻，我们会建立一个新的事务以记录未来开始的任何文件系统操作。任何现有的、不完整的操作仍然会使用现有的事务：我们不能在多个事务上拆分单个文件系统操作！
2. 开始将事务刷新到磁盘。在一个单独的log-writer内核线程的上下文中，我们开始向日志写入所有被事务修改过的元数据缓冲区。在这个阶段，我们还必须写入任何依赖数据（参见上面的部分：事务解剖）。
3. 提交缓冲区后，将其标记以固定事务，直到它不再脏（它已通过通常的写回机制写回主存储）。
4. 等待此事务中所有未完成的文件系统操作完成。我们可以在所有操作完成之前安全地开始写日志，允许这两个步骤在某种程度上重叠会更快。
5. 等待所有未完成的事务更新完全记录在日志中。
6. 更新日志头块以记录日志的新头部和尾部，将事务提交到磁盘。space released in the journal can now be reused by a later transaction. 
7. 当我们将事务的更新缓冲区写到日志中时，我们将它们标记以将事务固定在日志中。只有当这些缓冲区已同步到磁盘上的主缓冲区时，它们才会解除固定。只有当事务的最后一个缓冲区取消固定时，我们才能重用事务占用的日志块。当发生这种情况时，写入另一组日志头，记录日志尾部的新位置。日志中释放的空间现在可以由以后的事务重用。

# 事务间冲突

为了提高性能，我们在提交事务时不会完全暂停文件系统更新。相反，我们创建一个新的复合事务，在其中记录提交旧事务时到达的更新。

这就留下了一个问题，如果一个更新想要访问被另一个更新所占有的元数据缓冲区，而另一个更新包含于当前正在提交的旧事务，此时该怎么办。为了提交旧事务，我们需要将其缓冲区写入日志，但是我们不能在日志中写入任何不属于事务的更改，因为这将导致我们提交不完整的更新。

如果新事务只想读取有问题的缓冲区，那么没有问题：我们已经在两个事务之间创建了读/写依赖关系，但是由于复合事务总是以严格的顺序提交，我们可以安全地忽略冲突。

如果新事务想要写入缓冲区，事情就比较复杂了，我们需要缓冲区的旧副本来提交第一个事务，但是我们不能让新事务在不让它修改缓冲区的情况下继续进行。

这里的解决方案是在这种情况下创建缓冲区的新副本。将一份副本提供给新事务以进行修改。另一个由旧事务保留，并将像往常一样提交到日志。一旦事务提交，此副本将被删除。当然，在文件系统中的其他地方安全地记录此缓冲区之前，我们无法回收旧事务的日志空间，但由于必须将缓冲区提交到下一个事务的日志记录中，这一点会自动得到处理。

# 项目现状和未来的工作

这仍然是一项正在进行的工作。初始实现的设计既稳定又简单，我们不期望为了完成实现而需要对设计进行任何重大修改。

上述设计相对简单，只需对现有ext2fs代码进行少量修改，即可处理日志文件的管理、缓冲区和事务之间的关联以及不干净关闭后的文件系统恢复。

一旦我们有了一个稳定的代码库来测试，我们可以在许多可能的方向上扩展基本设计。最重要的是文件系统性能的调优。这将要求我们研究日志系统中任意参数的影响，如提交频率和日志大小。它还将涉及瓶颈研究，以确定是否可以通过修改系统设计来提高性能，并且已经有几个可能的设计扩展。

一个研究领域可能是考虑压缩更新中的日志更新。目前的方案要求我们向日志写入整个元数据块，即使块中只有一个比特被修改。我们可以通过只记录缓冲区中更改的值而不是记录整个缓冲区来非常容易地压缩这些更新。然而，目前还不清楚这是否会带来任何重大的性能优势。目前的方案对大多数写入来说不需要内存到内存的拷贝，这在CPU和总线利用率方面是一个巨大的性能优势。写入整个缓冲区产生的IO开销很低——因为更新是连续的，在现代磁盘IO系统中，它们直接从主存储器传输到磁盘控制器，而不经过缓存或CPU。

另一个重要的可能扩展领域是对快速NFS服务器的支持。NFS设计允许客户端在服务器崩溃时正常恢复：客户端将在服务器重新启动时重新连接。如果发生这种崩溃，服务器尚未安全写入磁盘的任何客户端数据都将丢失，因此NFS要求服务器在将客户端的文件系统请求提交到服务器磁盘之前，不得确认该请求已完成。

对于通用文件系统来说，这可能是一个难以支持的特性。NFS服务器的性能通常通过对客户端请求的响应时间来衡量，如果这些响应必须等待文件系统更新与磁盘同步，则总体性能会受到磁盘上文件系统更新延迟的限制。这与文件系统的大多数其他用途不同，在文件系统中，性能是根据缓存内更新的延迟而不是磁盘上更新的延迟来衡量的。

有些文件系统是专门设计来解决这个问题的。WAFL是一个基于事务树的文件系统，它可以在磁盘上的任何地方写入更新，但是Calaveras文件系统通过使用类似于上面建议的日志来达到同样的目的。不同之处在于，Calaveras将每个应用程序的文件系统请求在日志中记录为一个单独的事务，从而尽可能快地在磁盘上完成单独的更新。建议的ext2fs日志记录中的批处理提交牺牲了快速提交，而倾向于一次提交多个更新，从而以延迟为代价获得吞吐量(由于缓存的影响，磁盘上的延迟对应用程序是隐藏的)。

ext2fs日志记录有两种方式更适合在NFS服务器上使用，一种是使用较小的事务，另一种是记录文件数据和元数据。通过调整提交到日志的事务的大小，我们可能能够显著提高提交单个更新的周转时间。NFS还要求尽快将数据写入提交到磁盘，原则上没有理由不扩展日志文件以覆盖正常文件数据的写入。

最后，值得注意的是，这个方案中没有任何东西会阻止我们在几个不同的文件系统中共享一个日志文件。允许多个文件系统被记录到完全为此目的保留的单独磁盘上的日志中不需要太多额外的工作，并且在有许多日志文件系统都经历高负载的情况下，这可能会大大提高性能。单独的日志磁盘将几乎完全按顺序写入，因此可以保持高吞吐量，而不会损害主文件系统磁盘上的可用带宽。

# 结论

本文中概述的文件系统设计应该比Linux上现有的ext2fs文件系统提供显著的优势。它应该通过使文件系统在崩溃后更可预测和更快地恢复来提高可用性和可靠性，并且在正常操作中不应该导致太多的性能损失。

对日常性能最重要的影响是，新创建的文件必须快速同步到磁盘，以便将创建的文件提交到日志，而不是允许内核通常支持的数据延迟写回。这可能使日志文件系统不适合在***/tmp***文件系统上使用。

设计应该只需要对现有的ext2fs代码库进行最小的更改:大多数功能都是由新的日志机制提供的，该机制将通过一个简单的事务缓冲区IO接口与ext2fs主代码交互。

最后，这里介绍的设计构建在现有ext2fs磁盘上文件系统布局的基础上，因此可以在现有ext2fs文件系统中添加事务日志，无需重新格式化文件系统就可使用这些新特性。