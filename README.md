import { NextResponse } from "next/server"
import { getServerSession } from "next-auth/next"
import { authOptions } from "@/lib/auth"
import { applyVideoEdits, createGif } from "@/lib/advanced-video-editing"
import { getUserSubscription } from "@/lib/subscription-service"
import ffmpeg from "fluent-ffmpeg"
import * as tf from "@tensorflow/tfjs-node"
import * as cocoSsd from "@tensorflow-models/coco-ssd"
import OpenAI from "openai"
import { Server } from "socket.io"

const openai = new OpenAI({ apiKey: "your-openai-api-key" })

// Initialize WebSocket server for real-time collaboration
const io = new Server(3000, { cors: { origin: "*" } })

io.on("connection", (socket) => {
  console.log("User connected:", socket.id)

  socket.on("edit-video", (data) => {
    // Broadcast edits to all collaborators
    socket.broadcast.emit("video-updated", data)
  })

  socket.on("disconnect", () => {
    console.log("User disconnected:", socket.id)
  })
})

// Function to auto-crop video based on object detection
async function autoCropVideo(videoPath: string, outputPath: string) {
  const model = await cocoSsd.load()
  const framePath = "frame.jpg"

  await new Promise((resolve, reject) => {
    ffmpeg(videoPath)
      .screenshots({
        timestamps: [0],
        filename: framePath,
        folder: "./",
      })
      .on("end", resolve)
      .on("error", reject)
  })

  const image = await tf.node.decodeImage(framePath)
  const predictions = await model.detect(image)

  const mainObject = predictions.find((p) => p.class === "person")
  if (mainObject) {
    const [x, y, width, height] = mainObject.bbox

    await new Promise((resolve, reject) => {
      ffmpeg(videoPath)
        .videoFilter(`crop=${width}:${height}:${x}:${y}`)
        .output(outputPath)
        .on("end", resolve)
        .on("error", reject)
        .run()
    })
  }
}

// Function to generate video from text using AI
async function generateVideoFromText(text: string, outputPath: string) {
  const response = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [{ role: "user", content: text }],
  })

  const script = response.choices[0].message.content
  const ttsUrl = `https://text-to-speech-service.com/api?text=${encodeURIComponent(script)}`

  await new Promise((resolve, reject) => {
    ffmpeg()
      .input("background.mp4")
      .input(ttsUrl)
      .outputOptions(["-c:v copy", "-c:a aac", "-shortest"])
      .output(outputPath)
      .on("end", resolve)
      .on("error", reject)
      .run()
  })
}

// Function to generate AI-based thumbnail
async function generateThumbnail(videoPath: string, outputPath: string) {
  const framePath = "frame.jpg"
  await new Promise((resolve, reject) => {
    ffmpeg(videoPath)
      .screenshots({
        timestamps: [5],
        filename: framePath,
        folder: "./",
      })
      .on("end", resolve)
      .on("error", reject)
  })

  const model = await cocoSsd.load()
  const image = await tf.node.decodeImage(framePath)
  const predictions = await model.detect(image)

  const interestingObject = predictions.find((p) => p.class === "person" || p.class === "car")
  if (interestingObject) {
    const [x, y, width, height] = interestingObject.bbox

    await new Promise((resolve, reject) => {
      ffmpeg(framePath)
        .videoFilter(`crop=${width}:${height}:${x}:${y}`)
        .output(outputPath)
        .on("end", resolve)
        .on("error", reject)
        .run()
    })
  }
}

export async function POST(request: Request) {
  try {
    const session = await getServerSession(authOptions)

    if (!session) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
    }

    const subscriptionTier = await getUserSubscription(session.user.id)

    if (subscriptionTier !== "premium" && subscriptionTier !== "team") {
      return NextResponse.json(
        { error: "Advanced video editing is a premium feature. Please upgrade your subscription." },
        { status: 403 },
      )
    }

    const { videoPath, options } = await request.json()

    if (!videoPath) {
      return NextResponse.json({ error: "Video path is required" }, { status: 400 })
    }

    let editedVideoPath = videoPath

    // Apply auto-crop if requested
    if (options.autoCrop) {
      const croppedVideoPath = `cropped-${Date.now()}.mp4`
      await autoCropVideo(videoPath, croppedVideoPath)
      editedVideoPath = croppedVideoPath
    }

    // Generate video from text if requested
    if (options.generateFromText) {
      const generatedVideoPath = `generated-${Date.now()}.mp4`
      await generateVideoFromText(options.text, generatedVideoPath)
      editedVideoPath = generatedVideoPath
    }

    // Apply video edits
    editedVideoPath = await applyVideoEdits(editedVideoPath, options)

    // Create GIF if requested
    let gifPath = null
    if (options.createGif) {
      gifPath = await createGif(editedVideoPath, {
        start: options.gifStart || 0,
        duration: options.gifDuration || 3,
        fps: 10,
        width: 320,
      })
    }

    // Generate thumbnail if requested
    let thumbnailPath = null
    if (options.generateThumbnail) {
      thumbnailPath = `thumbnail-${Date.now()}.jpg`
      await generateThumbnail(editedVideoPath, thumbnailPath)
    }

    return NextResponse.json({
      success: true,
      editedVideoPath,
      gifPath,
      thumbnailPath,
    })
  } catch (error) {
    console.error("Error editing video:", error)
    return NextResponse.json({ error: "Failed to edit video", details: (error as Error).message }, { status: 500 })
  }
}
